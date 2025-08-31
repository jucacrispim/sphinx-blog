Apresentando: MongoMotor
========================

.. post:: Aug 17, 2018
   :category: computisses
   :author: Juca Crispim

Oi, pessoal. Tudo certo?

A ideia hoje é falar um pouquinho sobre o MongoMotor, que é uma biblioteca para
acesso assíncrono ao MongoDB usando Python.


O que é esse tal de assíncrono mesmo?
-------------------------------------

Em geral, operações de entrada e saída de dados (io) consomem bastante tempo
aguardando a chegada/envio de dados e enquanto um programa aguarda estas
operações todo o processamento é bloqueado e nada mais será feito até que os
dados sejam recebidos ou enviados. Para lidar com esta situação, as soluções
comuns são threads e multi-processos (vide apache) ou io assíncrono,
utilizando eventos do sistema operacional (vide nginx).

Com io assíncrono, quando executamos alguma operação de io ao invés de
aguardarmos o retorno, a operação é "deixada de lado" (num scheduler),
liberando o código para processar outras coisas, e quando obtivermos alguma
resposta da operação de io o sistema operacional enviará um evento informando
que a resposta chegou e assim o processamento da operação original pode ser
retomado.

Em Python, há muito tempo se tem projetos que usam a ideia de io assíncrono
para resolver este problema, como o Tornado ou o Twisted, mas com a chegada do
módulo asyncio à biblioteca padrão no Python 3.4 e com a inclusão da super
simpática sintaxe async/await no Python 3.5, operações de io assíncrono se
tornaram uma coisa muito mais corriqueira na linguagem.

Para uma super palestra sobre concorrência, veja este vídeo.
E esse async/await, hem?

Bem resumidamente, com async podemos definir corotinas em Python. Corotinas
são funções que quando chamadas retornam um objeto Future e precisam ter a sua
execução agendada por algum io loop. Algo assim:

.. code-block:: python

   async def my_coro():
       """Uma corotina que não faz nada"""

       # do something
       return 'some result'

   async def other_coro():
       """Uma corotina que chama outra corotina"""

       # r será o valor retornado por my_coro depois que sua execução for terminada.
       # Se chamássemos `my_coro()` sem `await` r seria uma future.
       r = await my_coro()
       # faça algo com r
       return 'a new result'

   # Agora agendamos a future retornada `other_coro()`
   loop = asyncio.get_event_loop()
   loop.run_until_complete(other_coro())

Para mais informações veja:
`asyncio docs <https://docs.python.org/3/library/asyncio.html#module-asyncio>`_
e leia esse
`post <https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/>`_.


Finalmente o MongoMotor
-----------------------

O mongomotor tira proveito do `Motor <https://motor.readthedocs.io/>`_, um
driver assíncrono para MongoDB, e do MongoEngine, com sua api à la Django
ORM, criando assim uma biblioteca bem simples para acesso assíncrono ao
mongodb. Basta criar classes representando seus documentos, declarar alguns
atributos - se quiser - e é isso, já podemos acessar o mongodb de maneira
assíncrona.


Código. Aleluia!
----------------

Acabou a parte chata e chegou o que todo mundo queria: código. Neste exemplo,
vamos analisar as perguntas mais recentes no stackoverflow.

A primeira coisa que precisamos fazer é instalar o mongomotor. Isto é feito
usando-se o pip. Num terminal digite o seguinte:

.. code-block::

   $ pip install mongomotor

Agora, com o mongomotor já instalado podemos começar a escrever o código.
Num arquivo python, faça:

.. code-block:: python

   # -*- coding: utf-8 -*-

   # A função connect é usada para connectar a um banco de dados
   from mongomotor import connect

   # Conectamos uma vez quando nosso programa começa e está feia a conexão.

   # Usando connect() sem parâmetros, o mongomotor vai tentar se conectar ao
   # mongo em localhost na porta 27017, o padrão para a instalação.
   connect()

   # Se necessário é possível passar outros parâmetros, além dos parâmetros de
   # autenticação
   connect(host='my.mongo.host', port=1234, username='myself', password='my-password')

   Agora vamos definir os nossos documentos:

   # A classe Document é a base para os nossos documentos que serão definidos
   from mongomotor import Document

   # Apesar de o mongo ser um banco de dados sem schema, usamos estes campos
   # para declarar nosso schema ficando mais fácil o entendimento posterior
   # do código.

   # Documentos com campos dinâmicos podem ser criados Usando-se a classe
   # mongomotor.DynamicDocument

   from mongomotor.fields import URLField, StringField, ListField, ReferenceField, IntField


   class Usuario(Document):
       """Um usuário que fez uma pergunta no stackoverflow."""

       # Este campo será um inteiro e é obrigatório, por isso o uso do
       # parâmetro required=True.
       # Usamos também o parâmetro unique=True para garantir que só exista
       # um documento com este valor.
       external_id = IntField(required=True, unique=True)
       """O id do usuário no so."""

       nome = StringField()
       """O nome usuário que será exibido. O nome, não o usuário. :P"""

       reputacao = IntField()
       """A reputação do usuário no site."""


   class Pergunta(Document):

       external_id = IntField(required=True, unique=True)
       """O id da pergunta no so."""

       titulo = StringField(required=True)
       """O título da pergunta"""

       # URLField é uma string que será validada para verificar se é uma
       # url
       url = URLField(required=True, unique=True)
       """A url da pergunta no so."""

       # ReferenceField aponta para um outro documento.
       # NOTA: Esta relação é feita na apliação, não no mongodb server.
       usuario = ReferenceField(Usuario, required=True)
       """O usuário que fez a pergunta."""

       # ListField indica que o campo é uma lista. Neste caso teremos uma lista
       # de strings.
       tags = ListField(StringField())
       """A lista de tags da pergunta"""


   # Para que o unique funcione precisamos criar os índices nas coleções.
   Usuario.ensure_indexes()
   Pergunta.ensure_indexes()

Pronto, nossos documentos já estão definidos. Para informações sobre todas as
opções para definir documentos, veja aqui.

Agora podemos inserir dados e fazer buscas nos documentos. Para inserir os
dados vamos user a api do stackoverflow.  Para fazer requisições http
assíncronas, usaremos a biblioteca aiohttp. Num terminal instale-a com:

.. code-block:: sh

   $ pip install aiohttp

Aqui a função para baixar os dados.

.. code-block:: python

   import json
   # Usamos aiohttp para fazer requests http assíncronos
   from aiohttp import ClientSession

   SO_URL = 'https://api.stackexchange.com/2.2/questions?order=desc&sort=activity&site=stackoverflow'


   async def get_so_questions():
       async with ClientSession() as session:
	   async with session.get(SO_URL) as response:
	       r = await response.read()

       # Retorna uma lista de dicionários. Cada dicionário contém informação
       # sobre uma pergunta.
       return json.loads(r.decode())['items']

O uso do aiohttp não está no escopo deste artigo, mas a ideia aqui é fazer
as operações de io (no caso os requests http) de maneira assíncrona.

Agora já podemos cadastrar alguns dados. Primeiro vamos criar um método para
criar um usuário baseado na informação retornada pela api.

.. code-block:: python

   class Usuario(Document):
       """Um usuário que fez uma pergunta no stackoverflow."""

       # Este campo será um inteiro e é obrigatório, por isso o uso do
       # parâmetro required=True.
       # Usamos também o parâmetro unique=True para garantir que só exista
       # um documento com este valor.
       external_id = IntField(required=True, unique=True)
       """O id do usuário no so."""

       nome = StringField()
       """O nome de usuário que será exibido"""

       reputacao = IntField()
       """A reputação do usuário no site."""

       # Adicionamos este método para inserir usuários.
       @classmethod
       async def get_or_create(cls, user_info):
	   """Retorna um usuário. Tenta obter um usuário através de sua
	   external_id. Se não existir, cria um novo usuário.

	   :param user_info: Um dicionário com informações do usuário enviado
	     pela api.
	   """

	   external_id = user_info['user_id']
	   nome = user_info['display_name']
	   reputacao = user_info['reputation']
	   try:
	       # O atributo `objects` é um objeto do tipo QuerySet.
	       # O método `get()` retorna um documento baseado nos parâmetros
	       # passados a este método.
	       user = await cls.objects.get(external_id=external_id)
	   except cls.DoesNotExist:
	       # Quando nenhum documento que se enquadra nos parâmetros
	       # é encontrado uma exceção `DoesNotExist` é levantada.
	       # Aqui neste caso criamos um novo documento
	       user = cls(external_id=external_id, nome=nome,
			  reputacao=reputacao)
	       # E salvamos o documento usando o método `save()`
	       await user.save()

	   return user

Agora vamos escrever um pouco de código para inserir as perguntas baseado no
retorno da api

.. code-block:: python

   class Pergunta(Document):
       """Uma pergunta feita no stackoverflow."""

       external_id = IntField(required=True, unique=True)
       """O id da pergunta no so."""

       titulo = StringField(required=True)
       """O título da pergunta"""

       # URLField é uma string que será validada para verificar se é uma
       # url
       url = URLField(required=True, unique=True)
       """A url da pergunta no so."""

       # ReferenceField aponta para um outro documento.
       # NOTA: Esta relação é feita na apliação, não no mongodb server.
       usuario = ReferenceField(Usuario, required=True)
       """O usuário que fez a pergunta."""

       # ListField indica que o campo é uma lista. Neste caso teremos uma lista
       # de strings.
       tags = ListField(StringField())
       """A lista de tags da pergunta"""


       @classmethod
       async def adicionar_perguntas(cls, perguntas):
	   """Adiciona as perguntas retornadas pela api.

	   :param perguntas: Uma lista de dicionários, cada um com informações
	     sobre uma pergunta.
	   """

	   # Lista para armazenar as perguntas a medida em que formos criando
	   # os documentos para salvá-las todas de uma vez só.
	   instancias = []
	   for pinfo in perguntas:
	       # Primeiro criamos usuário
	       usuario = await Usuario.get_or_create(pinfo['owner'])

	       # Agora criamos a pergunta
	       external_id = pinfo['question_id']
	       url = pinfo['link']
	       title = pinfo['title']
	       tags = pinfo['tags']
	       pergunta = cls(external_id=external_id, url=url, titulo=title,
			      tags=tags, usuario=usuario)

	       # Adicionamos à lista de instâncias para serem salvas depois
	       instancias.append(pergunta)

	   # Agora salvamos todas as instâncias de uma vez só.
	   await cls.objects.insert(instancias)

E uma função pra juntar tudo e popular o banco de dados.

.. code-block:: python

   async def populate_db():
       """Função para popular o banco de dados com as últimas perguntas do
       stackoverflow.
       """

       # Vamos limpar tudo primeiro
       await Pergunta.drop_collection()
       await Usuario.drop_collection()

       # Agora cadastramos as perguntas mais recentes
       perguntas = await get_so_questions()
       await Pergunta.adicionar_perguntas(perguntas)


Bom, depois de inserir alguns dados no banco, vamos fazer buscas nestes dados.

.. code-block:: sh

   async def stats():
       """Função que mostra alguns dados obtidos através da api do stackoverflow.
       """

       # O método `count()` é usado para contar a quantidade de documentos
       # em um queryset.
       total_perguntas = await Pergunta.objects.count()
       total_usuarios = await Usuario.objects.count()

       print('Temos um total de {} perguntas de {} usuários diferentes\n'.format(
	   total_perguntas, total_perguntas))

       # Podemos usar o método `order_by()` para ordenar os resultados.
       # Note que não é preciso o uso de await quando estamos filtrando/ordenando
       # um queryset. A operação de io só é executada quando um documento for
       # necessário
       usuarios = Usuario.objects.order_by('-reputacao')

       # Usamos método `fisrt` para pegar o primeiro resultado do queryset.
       # Aqui sim é necessário o uso de await.
       usuario = await usuarios.first()

       print('O usuário com maior reputação é: *{}* com reputação {}'.format(
	   usuario.nome, usuario.reputacao))

       # Podemos usar o método `filter()` para filtrar os resultados de um
       # queryset
       fileterd_qs = Pergunta.objects.filter(usuario=usuario)
       # E podemos iterar sobre os resultados do queryset com `async for`
       print('As perguntas de *{}* são:'.format(usuario.nome))
       async for pergunta in fileterd_qs:
	   print('- {}'.format(pergunta.titulo))
	   print('  tags: {}'.format(', '.join(pergunta.tags)))

       print('')

       # Com o método `item_frequencies()` podemos contas as repetições
       # de items de listas em documentos de um queryset
       popular_tags = await Pergunta.objects.item_frequencies('tags')

       tags = sorted([(k, v) for k, v in popular_tags.items()],
		     key=lambda x: x[1], reverse=True)
       most_popular = tags[0]
       print('A tag mais popular é *{}* com {} perguntas'.format(most_popular[0],
								 most_popular[1]))

       # Podemos filtar um queryset com base em um item de uma lista, no
       # nosso exemplo, com uma tag
       print('As perguntas de *{}* são:'.format(most_popular[0]))
       async for pergunta in Pergunta.objects.filter(tags=most_popular[0]):
	   print('- {}'.format(pergunta.titulo))
	   print('  tags: {}'.format(', '.join(pergunta.tags)))

	   # Note que para acessar uma referência é necessário o uso
	   # de await
	   usuario = await pergunta.usuario
	   print('  usuario: {}'.format(usuario.nome))

       print('')

Para a documentação completa de como fazer buscas usando o mongomotor veja
`aqui <http://mongomotor.poraodojuca.net/guide/defining-documents.html>`_.

E é isso. Acabamos de conhecer o básico do mongomotor. Pra finalizar, vamos
fazer uma função que coloca tudo isso junto:

.. code-block:: python

   async def main():
       print('Populando o banco de dados...')
       await populate_db()
       print('Banco de dados populado!\n')
       await stats()

E por fim, colocar isso aqui no final do nosso arquivo:

.. code-block:: python

   if __name__ == '__main__':

       import asyncio
       loop = asyncio.get_event_loop()
       loop.run_until_complete(main())

E feito! Nosso script final ficou assim:

.. code-block:: python

   # -*- coding: utf-8 -*-

   import json
   from aiohttp import ClientSession
   from mongomotor import connect
   from mongomotor import Document
   from mongomotor.fields import (URLField, StringField, ListField, ReferenceField,
				  IntField)

   connect()

   SO_URL = 'https://api.stackexchange.com/2.2/questions?order=desc&sort=activity&site=stackoverflow'


   async def get_so_questions():
       async with ClientSession() as session:
	   async with session.get(SO_URL) as response:
	       r = await response.read()

       return json.loads(r.decode())['items']


   class Usuario(Document):
       """Um usuário que fez uma pergunta no stackoverflow."""

       # Este campo será um inteiro e é obrigatório, por isso o uso do
       # parâmetro required=True.
       # Usamos também o parâmetro unique=True para garantir que só exista
       # um documento com este valor.
       external_id = IntField(required=True, unique=True)
       """O id do usuário no so."""

       nome = StringField()
       """O nome de usuário que será exibido"""

       reputacao = IntField()
       """A reputação do usuário no site."""

       # Adicionamos este método para inserir usuários.
       @classmethod
       async def get_or_create(cls, user_info):
	   """Retorna um usuário. Tenta obter um usuário através de sua
	   external_id. Se não existir, cria um novo usuário.

	   :param user_info: Um dicionário com informações do usuário enviado
	     pela api.
	   """

	   external_id = user_info['user_id']
	   nome = user_info['display_name']
	   reputacao = user_info['reputation']
	   try:
	       # O atributo `objects` é um objeto do tipo QuerySet.
	       # O método `get()` retorna um documento baseado nos parâmetros
	       # passados a este método.
	       user = await cls.objects.get(external_id=external_id)
	   except cls.DoesNotExist:
	       # Quando nenhum documento que se enquadra nos parâmetros
	       # é encontrado uma exceção `DoesNotExist` é levantada.
	       #
	       # Aqui neste caso criamos um novo documento
	       user = cls(external_id=external_id, nome=nome,
			  reputacao=reputacao)
	       # E salvamos o documento usando o método `save()`
	       await user.save()

	   return user


   class Pergunta(Document):
       """Uma pergunta feita no stackoverflow."""

       external_id = IntField(required=True, unique=True)
       """O id da pergunta no so."""

       titulo = StringField(required=True)
       """O título da pergunta"""

       # URLField é uma string que será validada para verificar se é uma
       # url
       url = URLField(required=True, unique=True)
       """A url da pergunta no so."""

       # ReferenceField aponta para um outro documento.
       # NOTA: Esta relação é feita na apliação, não no mongodb server.
       usuario = ReferenceField(Usuario, required=True)
       """O usuário que fez a pergunta."""

       # ListField indica que o campo é uma lista. Neste caso teremos uma lista
       # de strings.
       tags = ListField(StringField())
       """A lista de tags da pergunta"""


       @classmethod
       async def adicionar_perguntas(cls, perguntas):
	   """Adiciona as perguntas retornadas pela api.

	   :param perguntas: Uma lista de dicionários, cada um com informações
	     sobre uma pergunta.
	   """

	   # Lista para armazenar as perguntas a medida em que formos criando
	   # os documentos para salvá-las todas de uma vez só.
	   instancias = []
	   for pinfo in perguntas:
	       # Primeiro criamos usuário
	       usuario = await Usuario.get_or_create(pinfo['owner'])

	       # Agora criamos a pergunta
	       external_id = pinfo['question_id']
	       url = pinfo['link']
	       title = pinfo['title']
	       tags = pinfo['tags']
	       pergunta = cls(external_id=external_id, url=url, titulo=title,
			      tags=tags, usuario=usuario)

	       # Adicionamos à lista de instâncias para serem salvas depois
	       instancias.append(pergunta)

	   # Agora salvamos todas as instâncias de uma vez só.
	   await cls.objects.insert(instancias)


   # Para que o unique funcione precisamos criar os índices nas coleções.
   Usuario.ensure_indexes()
   Pergunta.ensure_indexes()


   async def populate_db():
       """Função para popular o banco de dados com as últimas perguntas do
       stackoverflow.
       """

       # Vamos limpar tudo primeiro
       await Pergunta.drop_collection()
       await Usuario.drop_collection()

       # Agora cadastramos as perguntas mais recentes
       perguntas = await get_so_questions()
       await Pergunta.adicionar_perguntas(perguntas)


   async def stats():
       """Função que mostra alguns dados obtidos através da api do stackoverflow.
       """

       # O método `count()` é usado para contar a quantidade de documentos
       # em um queryset.
       total_perguntas = await Pergunta.objects.count()
       total_usuarios = await Usuario.objects.count()

       print('Temos um total de {} perguntas de {} usuários diferentes\n'.format(
	   total_perguntas, total_perguntas))

       # Podemos usar o método `order_by()` para ordenar os resultados.
       # Note que não é preciso o uso de await quando estamos filtrando/ordenando
       # um queryset. A operação de io só é executada quando um documento for
       # necessário
       usuarios = Usuario.objects.order_by('-reputacao')

       # Usamos método `fisrt` para pegar o primeiro resultado do queryset.
       # Aqui sim é necessário o uso de await.
       usuario = await usuarios.first()

       print('O usuário com maior reputação é: *{}* com reputação {}'.format(
	   usuario.nome, usuario.reputacao))

       # Podemos usar o método `filter()` para filtrar os resultados de um
       # queryset
       fileterd_qs = Pergunta.objects.filter(usuario=usuario)
       # E podemos iterar sobre os resultados do queryset com `async for`
       print('As perguntas de *{}* são:'.format(usuario.nome))
       async for pergunta in fileterd_qs:
	   print('- {}'.format(pergunta.titulo))
	   print('  tags: {}'.format(', '.join(pergunta.tags)))

       print('')

       # Com o método `item_frequencies()` podemos contas as repetições
       # de items de listas em documentos de um queryset
       popular_tags = await Pergunta.objects.item_frequencies('tags')

       tags = sorted([(k, v) for k, v in popular_tags.items()],
		     key=lambda x: x[1], reverse=True)
       most_popular = tags[0]
       print('A tag mais popular é *{}* com {} perguntas'.format(most_popular[0],
								 most_popular[1]))

       # Podemos filtar um queryset com base em um item de uma lista, no
       # nosso exemplo, com uma tag
       print('As perguntas de *{}* são:'.format(most_popular[0]))
       async for pergunta in Pergunta.objects.filter(tags=most_popular[0]):
	   print('- {}'.format(pergunta.titulo))
	   print('  tags: {}'.format(', '.join(pergunta.tags)))

	   # Note que para acessar uma referência é necessário o uso
	   # de await
	   usuario = await pergunta.usuario
	   print('  usuario: {}'.format(usuario.nome))

       print('')


   async def main():
       print('Populando o banco de dados...')
       await populate_db()
       print('Banco de dados populado!\n')
       await stats()


   if __name__ == '__main__':

       import asyncio
       loop = asyncio.get_event_loop()
       loop.run_until_complete(main())


O script pode ser baixado aqui.

Salve isto em um arquivo chamado mmso.py e num terminal execute:

.. code-block:: sh

   $ python mmso.py

E voilà, eis a saída do nosso programa:


.. code-block:: sh

   Populando o banco de dados...
   Banco de dados populado!

   Temos um total de 30 perguntas de 30 usuários diferentes

   O usuário com maior reputação é: *jww* com reputação 49941
   As perguntas de *jww* são:
   - Undefined reference to symbol during link GCC inline assembly
     tags: c++, gcc, linker-errors, inline-assembly

   A tag mais popular é *python* com 5 perguntas
   As perguntas de *python* são:
   - BeautifulSoup4 findChildren() is empty
     tags: python, html, parsing, beautifulsoup
     usuario: Meghan M.
   - How to update weights in neural networks?
     tags: python, neural-network
     usuario: Absolute Idiot
   - assign results to dummy variable _
     tags: python, python-3.x
     usuario: cs0815
   - use opencv, cv2.videocapture in kivy with android - python for android
     tags: android, python, opencv, kivy, buildozer
     usuario: Vajira Prabuddhaka
   - How to get the value of a Django Model Field object
     tags: python, django, django-models
     usuario: Hugo Luis Villalobos Canto

Para informações mais detalhadas sobre o mongomotor, veja a
`documentação <http://mongomotor.poraodojuca.dev>`_.

O mongomotor é software livre, sinta-se à vontade para contribuir. :)
