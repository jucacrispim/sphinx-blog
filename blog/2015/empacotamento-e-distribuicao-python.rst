Empacotamento e distribuição de projetos Python sem mistério
============================================================

.. post:: Mar 6, 2015
   :category: computisses
   :author: Juca Crispim


Oi, pessoal, tudo certo?  O assunto hoje é o empacotamento dos nossos projetos
Python, ou seja, como a gente faz pra distribuir o nosso código pra outras
pessoas, isso de uma maneira fácil pra nós que desenvolvemos e pra quem vai
instalar o programa. A ideia aqui é explicar o que precisa fazer pra o nosso
programa poder ser instalado via pip, assim facilitando a distribuição.


Começando
---------

O que a gente precisa fazer pra empacotar e distribuir nosso projeto é  bem
pouca coisa. Antes de começar a gente só precisa das dependências, o setuptools
e o pip. Instalando o pip o setuptools vem como dependência. No debian o
pacote chama python-pip, e até acho que vem por padrão, nos red hat é algo
assim também. Em outros OSs não sei como instalar, mas vai lá, instala rapidão.
Aproveita e instala o virtualenvwrapper que a gente vai usar pra testar as coisas.

Já instalou? Beleza, agora vamos lá.


Um exemplo simples
------------------

Pra começar vamos usar um exemplo bem simples, o nosso programa vai ser um
único módulo, ou seja, ,um único arquivo .py. Vamos criar um diretório vazio
pra gente trabalhar:

.. code-block:: sh

   $ mkdir ~/dist-teste
   $ cd ~/dist-teste

É aqui neste diretório que a gente vai criar módulo (um arquivo.py) que
queremos distribuir. Vamos criar um módulo chamado meusuperprograma (arquivo
meusuperprograma.py). Aqui tem que se tomar cuidado pra não escolher um nome
que já seja de algum módulo da biblioteca padrão do Python ou que já seja de
alguma outra biblioteca popular (uma busca no pypi.python.org ajuda). Então,
aqui está o nosso módulo de exemplo:

.. code-block:: python

   #-*- coding: utf-8 -*-
   # arquivo meusuperprograma.py

   import time


   def faz_algo_dahora():
       return time.time()


Tendo isto, um programa que é um único módulo, nossa estrutura de diretórios
ficou assim:

.. code-block:: sh

   dist-teste/

   `-- meusuperprograma.py

Já temos um programa, agora a gente precisa ajeitar as coisas pra distribuir.
Pra isso precisamos criar um arquivo chamado setup.py no mesmo nível de
diretório que o nosso módulo (no diretório ~/dist-teste). Neste arquivo que vão
estar as configurações para o empacotamento. Um setup.py básico seria assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   from setuptools import setup


   setup(name='meusuperprograma',  # aqui o nome do seu programa
	 version='0.1',  # a versão.
	 author='Eu Mesmo',
	 author_email='me@myplace.net',
	 # Esta url deveria ser a url para a documentação/código/site oficial do projeto.
	 url='http://meusuperprograma.org',
	 # Aqui uma lista dos módulos que compõe a sua distribuição.
	 # No nosso caso, um módulo só.
	 py_modules=['meusuperprograma'],
   )

Então, agora com o setup.py temos a seguinte estrutura de diretórios:

.. code-block:: sh

   dist-teste/

   |-- setup.py

   `-- meusuperprograma.py

Com isso já podemos distribuir nosso programa. Só precisamos subir nosso código
para o CheeseShop e todo mundo vai poder instalar com um simples pip install.
Legal, né? Mas peraí... O que é mesmo o CheeseShop, hem?


CheeseShop, o Python Package Index
----------------------------------

CheeseShop é o codinome secreto do Python Package Index, aquele carinha que
você encontra em https://pypi.python.org/pypi e tenho certeza que você já
conhece. Quando a gente instala um programa com pip install... é aí que o pip
vai procurar o programa. Além deste pypi, a gente ainda tem um pypi de teste à
nossa disposição, esse aqui: https://testpypi.python.org/pypi. Vai lá, se
registra (nos dois, são bases separadas) e volta aqui. Rápido.


Pronto? Beleza. Agora a gente vai configurar o pip pra usar as nossas
credenciais. No arquivo ~/.pypirc coloque o seguinte:

.. code-block:: sh

   [distutils]
   index-servers =
       pypi
       testpypi

   [pypi]
   username: ze
   password: ninguém

   [testpypi]
   username: ze
   password: ninguém
   repository: https://testpypi.python.org/pypi

E é isso. Já temos o nosso super programa pra distribuir, já temos nosso
arquivo de configuração da distribuição (o setup.py) e já estamos registrados
nos lugares pra onde queremos subir nosso código. Agora é só alegria.


Distribuindo nosso programa
---------------------------

Como essa é a primeira versão do nosso programa, a gente vai precisar registrar
nosso projeto. A gente faz isso com o comando register do setuptools. No
exemplo abaixo registraremos nosso programa no pypi de teste, por isso usaremos
o parâmetro -r testpypi para indicar que usaremos o repositório que está  com o
nome testpypi no nosso .pypirc. Se não usássemos este parâmetro, iriamos
registrar no pypi oficial. Então, pra registrar fica assim:

.. code-block:: sh

   $ python setup.py register -r testpypi

   running register
   running egg_info

   [ output cortado ]

   running check
   Registering meusuperprograma to https://testpypi.python.org
   Server response (200): OK

Agora que já registramos nosso programa, podemos fazer um release, isto é,
fazer o upload de uma versão do nosso código. A gente faz isso com os comandos
sdist e upload. O comando sdist cria uma distribuição com os nossos arquivos e
o upload envia este arquivo para o servidor escolhido. Novamente usaremos o
parâmetro -r testpypi.

.. code-block:: sh

   $ python setup.py sdist upload -r testpypi

   running sdist
   running egg_info
   [ output cortado ]
   warning: sdist: standard file not found: should have one of README, README.rst, README.txt
   running check
   [ output cortado ]

   Creating tar archive
   removing 'meusuperprograma-0.1' (and everything under it)
   running upload
   Submitting dist/meusuperprograma-0.1.tar.gz to https://testpypi.python.org/pypi
   Server response (200): OK

E é isso, temos nosso programa prontinho pra distribuir - apesar do warining
por causa da falta de README.


Testando nossa distribuição
---------------------------

Pra testar a nossa distribuição, criaremos um virtualenv e também criaremos um
diretório vazio para ser nosso diretório de trabalho nos testes. O diretório
vazio é para nada 'ficar no caminho' e atrapalhar nos testes. Então, vamos
criar as coisas primeiro:

.. code-block:: sh

   $ mkvirtualenv meusuperprogramaenv -p /usr/bin/python3.4
   [ output cortado ]
   $ mkdir ~/dir-limpo && cd ~/dir-limpo

Agora, vamos instalar nosso programa e testar pra ver se foi tudo instalado.
Repare que será usado o parâmetro --index-url para indicar que o pip deve
procurar pelo pacote no CheeseShop de teste.

.. code-block:: sh

   $ pip install meusuperprograma --index-url=https://testpypi.python.org/pypi

   [ output cortado ]

   Successfully installed meusuperprograma
   Cleaning up...

   $ python
   Python 3.4.2 (default, Oct  8 2014, 10:45:20)
   [GCC 4.9.1] on linux
   Type "help", "copyright", "credits" or "license" for more information.
   >>> import meusuperprograma
   >>> meusuperprograma.faz_algo_dahora()
   1417391400.7311318
   >>>

É isso, nosso programa foi instalado corretamente pelo pip. Mas ainda tem mais
coisas pra gente ver.


Um programa com packages
------------------------

O nosso primeiro exemplo foi bem simples, um programa com apenas um módulo,
mas agora nosso programa cresceu ao invés de um módulo temos dois, e pra
organizar tudo isso vamos colocá-los dentro de um package. Um package é
simplesmente um diretório que contém módulos python.

A estrutura do nosso programa com package ficou assim:

.. code-block:: sh

   dist-teste/
   |-- COPYING
   |-- meusuperprograma/
   |   |-- __init__.py
   |   |-- modulo_a.py
   |   `-- modulo_b.py
   |-- README
   `-- setup.py

Além de alterarmos a estrutura do nosso programa também incluímos um arquivo
README (info sobre o programa, docs etc) e um arquivo COPYING com a lincença
do programa. Aqui está o conteúdo dos nossos módulos.

Arquivo meusuperprograma/modulo_a.py:

.. code-block:: python

   #-*- coding: utf-8 -*-
   # arquivo meusuperprograma/modulo_a.py

   import time


   def faz_algo_dahora():
       return time.time()

Arquivo meusuperprograma/modulo_b.py

.. code-block:: python

   # -*- coding: utf-8 -*-
   # arquivo meusuperprograma/modulo_b.py

   import datetime


   def faz_algo_sensacional(timestamp):
       dt = datetime.datetime.fromtimestamp(timestamp)
       return dt.strftime('%H:%M:%S - %d/%m/%Y')

Arquivo meusuperprograma/__init__.py

.. code-block:: python

   # -*- coding: utf-8 -*-
   # arquivo meusuperprograma/__init__.py

   from meusuperprograma.modulo_a import faz_algo_dahora
   from meusuperprograma.modulo_b import faz_algo_sensacional


   def faz_algo_sensacionalmente_dahora():
       timestamp = faz_algo_dahora()
       datahora = faz_algo_sensacional(timestamp)
       return {'timestamp': timestamp,
	       'datahora': datahora}

E com isto, temos um programa com um package para distribuir. Vamos fazer
algumas alterações no nosso setup.py para darem conta da nova versão do nosso
programa.

.. code-block:: python

   #-*- coding: utf-8 -*-

   from setuptools import setup


   setup(name='meusuperprograma',  # aqui o nome do seu programa
	 version='0.2',  # temos que alterar a versão.
	 author='Eu Mesmo',
	 author_email='me@myplace.net',
	 url='http://meusuperprograma.org',
	 # ao invés de usarmos o parâmetro py_modules usamos
	 # o parâmetro packages.
	 packages=['meusuperprograma'],
	 # Vamos colocar também alguns classificadores. Estes classificadores
	 # não são obrigatórios, mas deus gosta mais de você quando você
	 # classifica seus programas.
	 # Você pode ver uma lista com todos os classificadores aqui:
	 # https://pypi.python.org/pypi?%3Aaction=list_classifiers
	 classifiers=[
	     'Development Status :: 3 - Alpha',
	     'Intended Audience :: Developers',
	     'License :: OSI Approved :: GNU General Public License (GPL)',
	     'Natural Language :: Portuguese',
	     'Operating System :: OS Independent',
	     'Programming Language :: Python :: 3',
	     'Programming Language :: Python :: 3.2',
	     'Programming Language :: Python :: 3.3',
	     'Programming Language :: Python :: 3.4',
	     'Topic :: Software Development :: Libraries :: Python Modules',
	 ],

   )

Assim, já podemos fazer o release desta nova versão do programa.

.. code-block:: sh

   $ cd ~/dist-teste
   $ python setup.py sdist upload -r testpypi

   running sdist
   running egg_info

     [ output cortado ]

   running check

   [ output cortado ]

   Submitting dist/meusuperprograma-0.2.tar.gz to https://testpypi.python.org/pypi
   Server response (200): OK

Agora, vamos testar esta distribuição da nova versão


Testando a distribuição com packages
------------------------------------

Vamos atualizar a versão do meusuperprograma que está instalado no nosso
virtualenv de teste e vamos ao diretório limpo para testar se foi mesmo
instalado corretamente. Note que vamos usar uma opção nova, o parâmetro
--upgrade que diz para o pip atualizar a versão caso já haja alguma instalada.
Não esqueça de ativar seu virtualenv antes de atualizar a versão.

.. code-block:: sh

   $ # ative o virtualenv se não estiver ativado
   $ workon meusuperprogramaenv
   $ cd ~/dir-limpo
   $ pip install meusuperprograma --index-url=https://testpypi.python.org/pypi --upgrade

     [ output cortado ]

   Successfully installed meusuperprograma
   Cleaning up...

   $ python
   Python 3.4.2 (default, Oct  8 2014, 10:45:20)
   [GCC 4.9.1] on linux
   Type "help", "copyright", "credits" or "license" for more information.
   >>> import meusuperprograma
   >>> meusuperprograma.faz_algo_sensacionalmente_dahora()
   {'timestamp': 1417399828.9924762, 'datahora': '00:10:28 - 01/12/2014'}
   >>>


E tudo certo, nosso programa com package foi instalado corretamente.

O nosso programa ficou tão legal, tão sensacionalmente dahora que a gente
decidiu criar um script para o nosso programa poder ser chamado diretamente
da linha de comando, como um programa qualquer que a gente usa.


Um programa com script
----------------------

Para o nosso programa ter um script que qualquer um pode usar da linha de
comando, simplesmente criaremos, no nosso root dir do programa, um diretório
chamado scripts e dentro deste diretório colocaremos o script que queremos que
os usuários executem, e no nosso caso será um script chamado meusuperprograma
(sem o .py mesmo).

A estrutura de diretórios do nosso programa com este novo script ficou assim:

.. code-block:: sh

   /home/juca/dist-teste
   |-- COPYING
   |-- meusuperprograma
   |   |-- __init__.py
   |   |-- modulo_a.py
   |   `-- modulo_b.py
   |-- README
   |-- scripts
   |   `-- meusuperprograma
   `-- setup.py

E este é o conteúdo do arquivo scripts/meusuperprograma

.. code-block:: python

   #!/usr/bin/env python
   #-*- coding: utf-8 -*-

   # arquivo scripts/meusuperprograma

   import sys
   from meusuperprograma.modulo_a import faz_algo_dahora
   from meusuperprograma.modulo_b import faz_algo_sensacional


   if __name__ == '__main__':
       if len(sys.argv) > 1:
	   timestamp = float(sys.argv[1])
       else:
	   timestamp = faz_algo_dahora()

       datahora = faz_algo_sensacional(timestamp)
       msg = "A data e hora para o timestamp {timestamp} é: {datahora}"
       print(msg.format(timestamp=timestamp, datahora=datahora))

E precisamos alterar também o nosso setup.py, mais uma vez. Aqui a versão
alterada do setup.py:

.. code-block:: python

   #-*- coding: utf-8 -*-

   from setuptools import setup


   setup(name='meusuperprograma',  # aqui o nome do seu programa
	 version='0.3',  # temos que alterar a versão.
	 author='Eu Mesmo',
	 author_email='me@myplace.net',
	 url='http://meusuperprograma.org',
	 # ao invés de usarmos o parâmetro py_modules usamos
	 # o parâmetro packages.
	 packages=['meusuperprograma'],
	 # aqui indicamos onde ficam os scripts que serão instalados
	 scripts=['scripts/meusuperprograma'],
	 # Vamos colocar também alguns classificadores. Estes classificadores
	 # não são obrigatórios, mas deus gosta mais de você quando você
	 # classifica seus programas.
	 # Você pode ver uma lista com todos os classificadores aqui:
	 # https://pypi.python.org/pypi?%3Aaction=list_classifiers
	 classifiers=[
	     'Development Status :: 3 - Alpha',
	     'Intended Audience :: Developers',
	     'License :: OSI Approved :: GNU General Public License (GPL)',
	     'Natural Language :: Portuguese',
	     'Operating System :: OS Independent',
	     'Programming Language :: Python :: 3',
	     'Programming Language :: Python :: 3.2',
	     'Programming Language :: Python :: 3.3',
	     'Programming Language :: Python :: 3.4',
	     'Topic :: Software Development :: Libraries :: Python Modules',
	 ],
   )

E vamos fazer o release de novo e depois testar.

.. code-block:: sh

   $ cd ~/dist-teste
   $ python setup.py sdist upload -r testpypi
   running sdist
   running egg_info

   [ output cortado ]

   running check

   [ output cortado ]

   creating dist
   Creating tar archive
   removing 'meusuperprograma-0.3' (and everything under it)
   running upload
   Submitting dist/meusuperprograma-0.3.tar.gz to https://testpypi.python.org/pypi
   Server response (200): OK

   $ workon meusuperprogramaenv
   $ cd ~/dir-limpo
   $ pip install meusuperprograma --index-url=https://testpypi.python.org/pypi --upgrade
   Downloading/unpacking meusuperprograma

   [ output cortado ]

   Successfully installed meusuperprograma
   Cleaning up...



Agora, depois de instalado, só testar nosso programa pela linha de comando

.. code-block:: sh

   $ meusuperprograma
   A data e hora para o timestamp 1417404128.7673662 é: 01:22:08 - 01/12/2014

   $ meusuperprograma 0
   A data e hora para o timestamp 0.0 é: 21:00:00 - 31/12/1969

   $ meusuperprograma -62135585612
   A data e hora para o timestamp -62135585612.0 é: 00:00:00 - 01/01/1

É isso aí, tudo certinho.


Um programa com dependências
----------------------------

O nosso programa ficou tão legal que vamos até fazer uma versão web pra ele.
E claro que a gente não vai fazer tudo na mão, vamos usar um framework,
no caso o flask. Pra instalar é fácil, um simples pip install:

.. code-block:: sh

   $ pip install flask

Com o flask instalado vamos criar um módulo para a nossa aplicação web e um
script para rodar esta aplicação.

Primeiro, o arquivo meusuperprograma/webapp.py com a aplicação flask.

.. code-block:: python

   # -*- coding: utf-8 -*-
   # arquivo meusuperprograma/webapp.py

   from flask import Flask, Response
   from meusuperprograma import faz_algo_sensacionalmente_dahora

   minhasuperapp = Flask('meusuperprograma.webapp')


   @minhasuperapp.route('/')
   def index():
       info = faz_algo_sensacionalmente_dahora()
       ret = """
       A data e hora atual é: {datahora}.<br/>
       O timestamp pra isso é: {timestamp}
   """
       return(Response(ret.format(datahora=info['datahora'],
				  timestamp=info['timestamp'])))

Agora o arquivo scripts/meusuperprogramaweb, que é o script para rodar nossa
aplicação flask.

.. code-block:: python

   #!/usr/bin/env python
   #-*- coding: utf-8 -*-

   from meusuperprograma.webapp import minhasuperapp


   if __name__ == '__main__':
       minhasuperapp.run()

Com estes novos arquivos, a estrutura de diretórios ficou assim:

.. code-block:: sh

   /home/juca/dist-teste
   |-- COPYING
   |-- meusuperprograma
   |   |-- __init__.py
   |   |-- modulo_a.py
   |   |-- modulo_b.py
   |   `-- webapp.py
   |-- README
   |-- scripts
   |   |-- meusuperprograma
   |   `-- meusuperprogramaweb
   `-- setup.py

E agora vamos novamente alterar o setup.py:

.. code-block:: python

   #-*- coding: utf-8 -*-

   from setuptools import setup


   setup(name='meusuperprograma',  # aqui o nome do seu programa
	 version='0.4',  # temos que alterar a versão.
	 author='Eu Mesmo',
	 author_email='me@myplace.net',
	 url='http://meusuperprograma.org',
	 # ao invés de usarmos o parâmetro py_modules usamos
	 # o parâmetro packages.
	 packages=['meusuperprograma'],
	 # aqui indicamos onde ficam os scripts que serão instalados
	 scripts=['scripts/meusuperprograma', 'scripts/meusuperprogramaweb'],
	 # aqui indicamos quais as dependências de instalação
	 install_requires=['flask'],

	 # Vamos colocar também alguns classificadores. Estes classificadores
	 # não são obrigatórios, mas deus gosta mais de você quando você
	 # classifica seus programas.
	 # Você pode ver uma lista com todos os classificadores aqui:
	 # https://pypi.python.org/pypi?%3Aaction=list_classifiers
	 classifiers=[
	     'Development Status :: 3 - Alpha',
	     'Intended Audience :: Developers',
	     'License :: OSI Approved :: GNU General Public License (GPL)',
	     'Natural Language :: Portuguese',
	     'Operating System :: OS Independent',
	     'Programming Language :: Python :: 3',
	     'Programming Language :: Python :: 3.2',
	     'Programming Language :: Python :: 3.3',
	     'Programming Language :: Python :: 3.4',
	     'Topic :: Software Development :: Libraries :: Python Modules',
	 ],
   )

E é isso, tudo pronto pra lançar e testar novamente. Perceba que na hora de
instalar a nova versão de meusuperprograma vamos usar o parâmetro
--extra-index-url ao invés do parâmetro --index-url, isto porque queremos que
primeiro seja buscado no cheese shop live e depois no de teste.

.. code-block:: sh

   $ python setup.py sdist upload -r testpypi

   running sdist
   running egg_info

   [ output cortado ]

   running check

   [ output cortado ]

   Creating tar archive
   removing 'meusuperprograma-0.4' (and everything under it)
   running upload
   Submitting dist/meusuperprograma-0.4.tar.gz to https://testpypi.python.org/pypi
   Server response (200): OK

   $ cd ~/dir-limpo
   $ pip install meusuperprograma --extra-index-url=https://testpypi.python.org/pypi --upgrade

   [ output cortado ]

   Successfully installed meusuperprograma
   Cleaning up...

Perceba que na hora da instalação foi instalado também, automaticamente, o
flask e suas dependências.

Agora, vamos testar nossa aplicação web.

.. code-block:: sh

   $ meusuperprogramaweb
    * Running on http://127.0.0.1:5000/

E abra seu browser e acesse http://127.0.0.1:5000/ para ver a nossa aplicação
web rodando.

Só que a aparência da aplicação web ficou meio xoxa, não? Vamos fazer um
template lindão pra melhorar as coisas

Um programa com package data
----------------------------

Os arquivos que não são arquivos python, mas que serão incluídos na
distribuição são chamados de package data, isso inclui o template (um arquivo
.html) que usaremos para a nossa aplicação web.

Então, primeiro fazemos um template bem bonitão, que vai ficar em
meusuperprograma/templates/template.html

.. code-block:: html

   <html>
     <head>
       <title>Meu Super Programa Versão Web!</title>
     </head>

     <body>
       <div> A data e hora atual é: <span>{{ datahora }}</span></div>
       <div> O timestamp pra isso é: <span>{{ timestamp }}</span></div>
     </body>
   </html>

E depois alteramos a nossa webapp pra que passe a usar o template:

.. code-block:: python

   # -*- coding: utf-8 -*-

   from flask import Flask, render_template
   from meusuperprograma import faz_algo_sensacionalmente_dahora


   minhasuperapp = Flask('meusuperprograma.webapp')


   @minhasuperapp.route('/')
   def index():
       info = faz_algo_sensacionalmente_dahora()
       contexto = {'datahora': info['datahora'],
		   'timestamp': info['timestamp']}

       return render_template('template.html', **contexto)

Já alteramos tudo o que precisávamos no nosso código, mas ainda precisamos
alterar o setup.py e criar mais um novo arquivo, que se chamará MANIFEST.in e
ficará na raiz do nosso projeto. Este arquivo MANIFEST.in é um arquivo onde
dizemos quais arquivos de package data devem ser incluídos na distribuição.
O nosso ficará assim:

.. code-block:: sh

   include meusuperprograma/templates/template.html

Simplesmente usamos a diretiva include para dizer qual arquivo deve ser
incluído na distribuição.

Com este novo arquivo, nossa estrutura de diretórios ficou assim:

.. code-block:: sh

   /home/juca/dist-teste
   |-- COPYING
   |-- MANIFEST.in
   |-- meusuperprograma
   |   |-- __init__.py
   |   |-- modulo_a.py
   |   |-- modulo_b.py
   |   |-- templates
   |   |   `-- template.html
   |   `-- webapp.py
   |-- README
   |-- scripts
   |   |-- meusuperprograma
   |   `-- meusuperprogramaweb
   `-- setup.py

Agora vamos alterar o setup.py. É uma alteração simples. Passaremos a usar
o parâmetro include_package_data=True para indicar que os arquivos não-python
devem ser incluídos. Com esta mudança nosso setup.py ficou assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   from setuptools import setup


   setup(name='meusuperprograma',  # aqui o nome do seu programa
	 version='0.4',  # temos que alterar a versão.
	 author='Eu Mesmo',
	 author_email='me@myplace.net',
	 url='http://meusuperprograma.org',
	 # ao invés de usarmos o parâmetro py_modules usamos
	 # o parâmetro packages.
	 packages=['meusuperprograma'],
	 # aqui indicamos onde ficam os scripts que serão instalados
	 scripts=['scripts/meusuperprograma', 'scripts/meusuperprogramaweb'],
	 # aqui indicamos quais as dependências de instalação
	 install_requires=['flask'],
	 # aqui dizemos que é para incluir os arquivos que não são
	 # arquivos python
	 include_package_data=True,

	 # Vamos colocar também alguns classificadores. Estes classificadores
	 # não são obrigatórios, mas deus gosta mais de você quando você
	 # classifica seus programas.
	 # Você pode ver uma lista com todos os classificadores aqui:
	 # https://pypi.python.org/pypi?%3Aaction=list_classifiers
	 classifiers=[
	     'Development Status :: 3 - Alpha',
	     'Intended Audience :: Developers',
	     'License :: OSI Approved :: GNU General Public License (GPL)',
	     'Natural Language :: Portuguese',
	     'Operating System :: OS Independent',
	     'Programming Language :: Python :: 3',
	     'Programming Language :: Python :: 3.2',
	     'Programming Language :: Python :: 3.3',
	     'Programming Language :: Python :: 3.4',
	     'Topic :: Software Development :: Libraries :: Python Modules',
	 ],
   )

E agora sim temos tudo pronto. Vamos gerar nossa distribuição e testar.

.. code-block:: sh

   $ python setup.py sdist upload -r testpypi

   running sdist
   running egg_info

     [output cortado]

   running check

     [output cortado]

   Creating tar archive
   removing 'meusuperprograma-0.5' (and everything under it)
   running upload
   Submitting dist/meusuperprograma-0.5.tar.gz to https://testpypi.python.org/pypi
   Server response (200): OK

   $ workon meusuperprogramaenv
   $ cd ../dir-limpo
   $ pip install meusuperprograma --extra-index-url=https://testpypi.python.org/pypi --upgrade

     [output cortado]

   $ meusuperprogramaweb
    * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)

E é isso. Agora só abrir seu navegador em 127.0.0.1:5000 que você vai ser seu
super programa versão web agora com um lindo template.

Tá vendo, agora não tem mais mistério em como distribuir seus projetos (puro)
python. Molezinha!

Dúvidas? Fiquem à vontade, podem mandar bala!
