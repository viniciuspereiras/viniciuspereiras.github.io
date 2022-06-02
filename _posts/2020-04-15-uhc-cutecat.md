---
layout: article
title:  "UHCCTF (Classificatória) - CuteCat 🐈 (Official Write-Up)"
tags: pt-br ctf write-up
---


Esse challenge foi feito para ser fofo e maluco ao mesmo tempo...
Meu primeiro challenge MISC.

# Part 1

Começamos o challenge com um .zip criptografado com senha:

![/assets/cutecat/Untitled.png](/assets/cutecat/Untitled.png)

Temos algumas opções para partir, ou rodar um bruteforce ou achar a senha.

Testando algumas coisas na mão acabei conseguindo acertar, a senha é o número que está no nome do arquivo, no caso a senha do [22800.zip](http://22800.zip) é 22800.

![/assets/cutecat/Untitled%201.png](/assets/cutecat/Untitled%201.png)

Extraindo, temos mais um arquivo, o [41992.zip](http://41992.zip), testando o mesmo método conseguimos deszipar.

![/assets/cutecat/Untitled%202.png](/assets/cutecat/Untitled%202.png)

Pelo jeito é um padrão que precisa ser quebrado, vamos desenvolver um script para pegar o nome do arquivo e fazer a extração em loop.

Existem MUITAS formas diferentes de resolver essa primeira parte do challenge, eu usarei shell script para automatizar o processo.

```bash
#!/bin/bash
mkdir data                           
echo 'any key + enter to start:'     
read key
while true                          
do
	name=$(ls | grep .zip)            
	name=${name%.zip}                 
	echo "Unziping: $name"     
	unzip -j -P $name $name.zip      
	status=$(echo $?)                  
	if [[ $status != 0 ]]              
	then
	echo "Error extracting $name.zip" 
	exit                              
	fi                                 
	mv $name.zip ./data               
done
```

Agora vou explicar o script, o que ele faz é em um loop dar um ls na pasta que só vai retornar no output os arquivos que terminarem com .zip, nesse output terá o nome do arquivo que precisamos e será salvo na variável name, então excluímos o ".zip" da nossa variável name e então extraímos o arquivo desse .zip e movemos o .zip de origem para a pasta "data", assim não teremos problemas no "ls" (o if no meio do código verifica se a extração foi feita com sucesso, se não ele mostra qual .zip deu erro e encerra o programa deixando o script que deu erro na pasta raiz). Assim deve funcionar, vamos testar...

![/assets/cutecat/Untitled%203.png](/assets/cutecat/Untitled%203.png)

![/assets/cutecat/Untitled%204.png](/assets/cutecat/Untitled%204.png)

# Part 2

Depois de um tempo ele terminou de extrair tudo mas deu erro no 46691.zip, e mostrou que dentro dele tem um arquivo chamado cutecat.jpg...

Depois de tentar umas senhas e não conseguir vamos fazer um bruteforce.

Vou utilizar o john para isso.

![/assets/cutecat/Untitled%205.png](/assets/cutecat/Untitled%205.png)

Rapidinho conseguimos a senha, usando ela conseguimos extrair o cutecat.jpg

# Part 3

Aparentemente o arquivo está corrompido não conseguimos visualizar a imagem.

![/assets/cutecat/Untitled%206.png](/assets/cutecat/Untitled%206.png)

![/assets/cutecat/Untitled%207.png](/assets/cutecat/Untitled%207.png)

A principio parece que a assinatura do arquivo pode estar errada, por isso o "file" do Linux não conseguiu identificar o tipo do arquivo. Vamos extrair o hexadecimal da imagem para poder analisar.

![/assets/cutecat/Untitled%208.png](/assets/cutecat/Untitled%208.png)

Os primeiros bytes dos arquivos costumam ser conhecidos como magic-bytes, vamos ver como é a assinatura (signature ou magic-bytes) de um arquivo jpg.

[List of file signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)

![/assets/cutecat/Untitled%209.png](/assets/cutecat/Untitled%209.png)

Nossa imagem:

![/assets/cutecat/Untitled%2010.png](/assets/cutecat/Untitled%2010.png)

A partir do terceiro byte a assinatura esta correta, então só teremos que trocar os 00 por ff da nossa imagem

Para isso também existem várias formas de fazer, vou usar um módulo do CyberChef para renderizar a imagem, chamado Render Image.

[CyberChef](https://gchq.github.io/CyberChef/)

![/assets/cutecat/Untitled%2011.png](/assets/cutecat/Untitled%2011.png)

Selecionando o módulo Render Image usando o input format de Hex eu copiei todo o conteudo do extract que fizemos anteriormente e alterei os dois primeiros bytes que estavam errados para "ff" e apareceu um gatinho fofinho, ownt! 

Vamos baixar a imagem!

![/assets/cutecat/Untitled%2012.png](/assets/cutecat/Untitled%2012.png)

![/assets/cutecat/Untitled%2013.png](/assets/cutecat/Untitled%2013.png)

Ownt!!!!!!!!!!!!!

Parece que temos uma senha, acredito que o único lugar que poderíamos usar essa senha seja se tiver esteganografia na imagem...

![/assets/cutecat/Untitled%2014.png](/assets/cutecat/Untitled%2014.png)

É isso, foi fofo né? 😸