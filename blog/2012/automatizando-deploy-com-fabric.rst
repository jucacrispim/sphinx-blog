.. post:: Jan 08, 2012
   :category: computisses
   :author: Juca Crispim


Automatizando o deploy com fabric
=================================

Fala, pessoal. Tudo certo? Hoje eu vim falar sobre o fabric. O fabric é um cara
que te ajuda a automatizar o deploy permitindo que você execute comandos de shell
na máquina local e (o que é mais legal) em um servidor remoto de maneira muito
simples.


Como assim?
-----------

Imagine que você tem seu programa lá, bonitão, works on my machine certified,
mas você precisa por isso em algum lugar acessível ao público. Você pode muito
bem gerar um .tar do seu código, copiar pro servidor, instalar... enfim, fazer
tudo o que precisa na mão. Nem é complicado. Mas fazer isso é chato, e se a
coisa cresce, tem sempre o risco de esquecer algo. É aí que entra o fabric! Com
ele, você escreve um script (em Python) pra fazer o seu deploy.


Tá, beleza. Mas como funciona?
------------------------------

É bem simples. A primeira coisa a se fazer é instalar o fabric e a maneira mais
fácil de fazer isso é pelo gerenciador de pacotes do seu sistema operacional
(seu s.o. tem gerenciador de pacotes, não?). Depois de instalado o fabric, é só
você criar um arquivo chamado fabfile.py contendo os comandos necessários ao
deploy.

Pra começar, vamos fazer um 'olá' com o fabric pra gente ver como funciona. O
fabfile pro nosso 'ola' ficou assim:

.. code-block:: python

   #-*- coding: utf-8 -*-
   # Arquivo fabfile.py

   def ola(nome):
       print 'olá, ', nome

A sintaxe pra se executar o fabfile é a seguinte: fab <nome_da_funcao>:<arg1>,
<arg2>, ...

Então, pra executar nosso fabfile acima, executamos o seguinte comando:

.. code-block:: sh

   $ fab ola:juca
   ola,  juca

   Done.


Tá, entendi. Agora um exemplo decente, vai.
-------------------------------------------

Agora que já vimos como usar o fabric, vamos a um exemplo real, pra gente dar
uma olhada em algumas coisas interessantes da api do fabric.

A idéia aqui vai ser a seguinte: Eu tenho um repositório git na minha máquina
contendo o código que eu quero subir. O procedimento pra subir é gerar um .tar
contendo o código de uma named tree qualquer (um branch, uma tag, um
commit...), copiar esse tar pro servidor remoto, desempacotar o tar no servidor
remoto, se já tiver uma versão mais antiga instalada, desinstalar essa versão
antiga, instalar a versão nova e por fim, reiniciar o web server.

Tudo muito simples, mas ficar fazendo isso é muito chato, então o fabfile
abaixo resolve isso pra gente:

.. code-block:: python

   import os
   import time

   # aqui importando uns caras legais da api do fabric
   # local - roda um comando de shell na máquina local
   # run - roda um comando de shell no servidor remoto
   # put - faz uma cópia via ssh (scp) pro servidor remoto
   # env - configurações do ambiente

   from fabric.api import local, run, put, env


   LOCAL_SRC_PATH = '/home/juca/mysrc/sourcecode2html'
   LOCAL_BUILD_PATH = '/tmp/amazon-build'
   REMOTE_SRC_PATH = '/home/deployuser/src/codeprettifier'

   # aqui é uma string do tipo usuario@host[:porta]
   # usuario é um usuário do sistema no servidor remoto.
   # É uma string como o que você passa pro ssh
   env.hosts = ['deployuser@myserver']

   # É essa função que vai ser chamada na execução do fabfile, algo como:
   # fab deploy:master
   def deploy(tree_name):
       """ Executa as ações necessárias ao deploy
       """

       tar_file = _package_named_tree(tree_name)
       remote_tar_path, filename = _send_file(tar_file)
       _unpack_code(remote_tar_path, filename)
       _uninstall_last_version()
       _create_link_to_lastest(remote_tar_path)
       _build()
       _install()
       _restart_server()

   def _package_named_tree(tree_name):
       """ cria um arquivo .tar.bz2 baseado numa named tree do git
       """
       try:
	   os.mkdir(LOCAL_BUILD_PATH)
       except OSError:
	   pass

       os.chdir(LOCAL_SRC_PATH)
       filename = '%s/codeprettifier-%s.tar.bz2' %(LOCAL_BUILD_PATH,
						   tree_name)

       pack_command = 'git archive %s --prefix=codeprettifier/ |' % tree_name
       pack_command += ' bzip2 > %s' % filename

       # aqui, executando o comando na máquina local
       # com o local() da api do fabric
       local(pack_command)
       return filename

   def _send_file(filename):
       """ send file to remote server
       """

       remote_tar_path = REMOTE_SRC_PATH + '/%s/' % int(time.time())
       try:
	   # Aqui executando run(). O mkdir aí em baixo vai ser
	   # executado no servidor remoto.
	   # Assim que o primeiro run() é chamado, vai ser perguntada
	   # a senha do usuário no host remoto.
	   run('mkdir -p %s' % remote_tar_path)
       except:
	   pass

       # aqui enviando arquivo via scp usando o put()
       # da api do fabric
       put(filename, remote_tar_path)
       filename = filename.split('/')[-1]
       return remote_tar_path, filename

   def _unpack_code(remote_tar_path, filename):
       """ Desempacota o código no servidor remoto
       """
       run('cd %s' % remote_tar_path)
       run('tar -xjvf %s/%s -C %s' % (remote_tar_path, filename, remote_tar_path))

   def _uninstall_last_version():
       """ unpacks the code on remote server
       """
       try:
	   run('cd %s/latest/codeprettifier' % REMOTE_SRC_PATH)
       except:
	   return

       uninstall_command = 'cd %s/latest/codeprettifier && ' % REMOTE_SRC_PATH
       uninstall_command += "sudo make uninstall | grep -v codeprettifier/ |"
       uninstall_command += 'grep -v Java/ | grep -v MultiLineStringDelimiter.pm |'
       uninstall_command += "cut -d'k' -f2 | grep -i CodePrettifier |"
       uninstall_command += " grep -v codeprettifier.pl |xargs sudo rm"

       try:
	   run(uninstall_command)
       except:
	   pass

       run('rm %s/latest' % REMOTE_SRC_PATH)

   def _create_link_to_lastest(remote_tar_path):
       run('ln -s %s %s/latest' % (remote_tar_path, REMOTE_SRC_PATH))

   def _build():
       """ Cria o Makefile pra instalação
       """
       remote_latest_dir = REMOTE_SRC_PATH + '/latest/codeprettifier'
       run('cd %s && perl Makefile.PL' % remote_latest_dir)

   def _install():
       """ Faz a instalação em si
       """
       remote_latest_dir = REMOTE_SRC_PATH + '/latest/codeprettifier'
       run('cd %s && sudo make install' % remote_latest_dir)

   def _restart_server():
       """ reinicia o server
       """
       run('sudo /sbin/service httpd restart')

Agora, é só executar

.. code-block:: sh

   $ fab deploy:master

   [deployuser@myserver] Executing task 'deploy'
   [localhost] local: git archive master --prefix=codeprettifier/ | bzip2 > /tmp/amazon-build/codeprettifier-master.tar.bz2
   [deployuser@myserver] run: mkdir -p /home/deployer/src/codeprettifier/1326011668/
   [deployuser@myserver] Login password:
   [deployuser@myserver] put: /tmp/amazon-build/codeprettifier-master.tar.bz2 -> /home/deployer/src/codeprettifier/1326011668/codeprettifier-master.tar.bz2
   [deployuser@myserver] run: cd /home/deployuser/src/codeprettifier/1326011668/
   [deployuser@myserver] run: tar -xjvf /home/deployuser/src/codeprettifier/1326011668//codeprettifier-master.tar.bz2 -C /home/deployuser/src/codeprettifier/1326011668/
   ...

   [deployuser@myserver] run: cd /home/deployuser/src/codeprettifier/latest/codeprettifier && sudo make uninstall | grep -v codeprettifier/ |grep -v Java/ | grep -v MultiLineStringDelimiter.pm |cut -d'k' -f2 | grep -i CodePrettifier | grep -v codeprettifier.pl |xargs sudo rm
   [deployuser@myserver] run: rm /home/deployuser/src/codeprettifier/latest
   [deployuser@myserver] run: ln -s /home/deployuser/src/codeprettifier/1326011668/ /home/deployuser/src/codeprettifier/latest
   [deployuser@myserver] run: cd /home/deployuser/src/codeprettifier/latest/codeprettifier && perl Makefile.PL
   ...

   [deployuser@myserver] run: cd /home/deployuser/src/codeprettifier/latest/codeprettifier && sudo make install
   ...

   [deployuser@myserver] run: sudo /sbin/service httpd restart
   ...

   Done.
   Disconnecting from deployuser@myserver... done.

E pronto, seu deploy foi feito automaticamente!

Pra finalizar, quero dizer que isso foi só um exemplo, você pode escrever o
procedimento de deploy que quiser com o fabric. Ele é bem versátil!

Bom, é isso pessoal. Até a próxima! :)
