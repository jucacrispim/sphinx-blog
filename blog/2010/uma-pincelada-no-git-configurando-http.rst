.. post:: Jan 8, 2010
   :tags: computisses
   :author: Juca Crispim

Uma pincelada no GIT parte II - Configurando HTTP
=================================================

NÃO LEIA ISSO - ou melhor, pode até ler, mas esse aqui não é o jeito certo de
fazer as coisas. :P Leia esse aqui

Antes de mais nada, a fonte primária de informação é essa aqui. Isto aqui é a
continuação do post anterior sobre o git. Como prometido, hoje vou explicar como
configurar o git para usar http ou invés de ssh.


Começando do começo
--------------------


Bom, começarmos a configurar o servidor você vai precisar de algumas coisas
instaladas nele. São elas:

- Um servidor web apache

- Git

Você vai precisar também ter acesso como root neste servidor para poder fazer
a configuração inicial do repositório. Antes de prosseguir, vale uma
ressalva: O apache é um treco chato... Se você pegar os fontes e compilar,
o 'document root' e os arquivos de configuração serão em um lugar. Se você
estiver usando um Red Hat serão em outro, em um Debian serão em outro ainda.
E no windows então! Nem faço idéia como funciona o apache no windows.

Por isso, primeiro sempre darei uma explicação "genérica" de como a coisa
funciona. Nos exemplos que darei, os caminhos de arquivos e tudo mais serão
baseados em Debian. Se você uma um sistema operacional diferente, leia o manual
do apache e veja onde estão localizados seus arquivos.


Configurando o servidor
-----------------------

A primeira coisa a fazer é certificar-se que o módulo dav_module está sendo
carregado pelo apache. Isto pode ser feito adicionando as seguitnes linhas os
seu httpd.conf:

.. code-block:: sh

   LoadModule dav_module libexec/httpd/libdav.so
   AddModule mod_dav.c
   DAVLockDB "/usr/local/apache2/temp/DAV.lock"

No Debian, isto pode ser feito de uma outra maneira. Deste jeito:

.. code-block:: sh

   # a2enmod dav_fs
   # a2enmod dav

Depois disto, é preciso criar um repositório "pelado" para o git. Para isto,
criaremos um diretório dentro do nosso "document root", criaremos o reporitório
dentro deste diretório recém criado, e por fim, deixaremos o usuário do apache
como dono deste diretório e de todos os seus sub-diretórios.
No Debian fica assim:

.. code-block:: sh

   # mkdir /var/www/novo_projeto.git
   # cd /var/www/novo_projeto.git
   # git --bare init
   # cd ../
   # chown -R www-data:www-data novo_projeto.git/

Agora, vamos criar um usuário/senha para acessar o projeto . A sintaxe é assim:
htpasswd -c /caminho/para/o/arquivo/de/senha <usuario>.

O diretório onde se encontra o arquivo de senhas tem que ser lido pelo apache
e, de preferência, não ser lido pelo resto do mundo. O parâmetro -c passado ao
htpasswd significa que criaremos um novo arquivo. Se o arquivo já exitir e
você quiser apenas acrescentar mais um usuário, htpasswd deve ser chamado sem o
parâmetro -c. Aqui no Debian, ficou assim:

.. code-block:: sh

   # htpasswd -c /etc/apache2/passwd.git gituser

Depois de criado o usuário, vamos definir as regras do apache para o diretório
do nosso repositório. Geralmente você precisaria adicionar algo assim ao seu
httpd.conf:

.. code-block:: sh

   <Location /meu_repo>
     DAV on    AuthType Basic
     AuthName "Git"
     AuthUserFile /caminho/para/o/arquivo/de/senha
     Require valid-user
   </Location>

Como o Debian lê automaticamente os arquivos em /etc/apache2/conf.d/, eu criei
o arquivo /etc/apache2/conf.d/git.conf e adicionei o seguite a ele:

.. code-block:: sh

   <Location /novo_projeto.git>
     DAV on    AuthType Basic
     AuthName "Git"
     AuthUserFile /etc/apache2/passwd.git
     Require valid-user
   </Location>

Agora, reinicie o apache. No Debian fica assim: # /etc/init.d/apache2 restart
Neste ponto o servidor já deve estar funcionando corretamente. Para testar,
acesse o seu servidor da seguinte maneira: http://servidor/novo_projeto.git

O servidor peguntará seu usuário/senha. Depois de informá-las, se você ver uma
listagem de diretórios e arquivos do git, o servidor está funcionando corretamente.


Configurando o cliente
----------------------

Agora que o servidor já está configurado, é hora de configurar o cliente. A
primeira coisa a fazer, é informar nosso usuário e senha pra que não tenhamos
que ficar digitando isso toda vez... é chato! Fazemos isso adicionando as
seguintes linhas ao arquivo $HOME/.netrc:

.. code-block:: sh

   machine <servidor> login <usuário> password <senha>

Como nossa senha está aí, é bom restringir o acesso a este arquivo. Fazemos
isto assim:

.. code-block:: sh

   $ chmod 600 ~/.netrc

Depois disso, precisamos configurar o git para acessar nosso servidor. Isso é
muito simples.

.. code-block:: sh

   $ git config remote.novo_projeto.url http://<usuário>@<servidor>/novo_projeto.git/

OBS: Não esqueça da '/' no final da url, senão você vai cair num
redirecionamento eterno...

Bom, aqui já está tudo configurado. Só o que precisamos fazer agora é
'empurrar' os arquivos do projeto para o servidor.

.. code-block:: sh

   $ git push novo_projeto master

   Fetching remote heads...
   refs/ refs/heads/ refs/tags/
   updating 'refs/heads/master' from 5b0bc00758855aef6dafe7aa9849443aea0dbf1c to b4857d97182a582c76961370de489b14385f9af9
   sending 13 objects done

Bom, é isso aí! Git configurado pra usar HTTP. Agora você que decide como usar seu Git.
