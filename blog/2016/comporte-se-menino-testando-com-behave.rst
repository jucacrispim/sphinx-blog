Comporte-se, menino! Testando suas aplicações com behave
========================================================

.. post:: Nov 21, 2016
   :tags: computisses
   :author: Juca Crispim

Boas, pessoal! Tudo tranquilo? Hoje vamos falar sobre como testar o
comportamento de suas aplicações, fazendo testes de aceitação, usando um
carinha chamado behave.


Testes de aceitação? Behave? WTF!?
----------------------------------

Testes de aceitação são testes que verificam a correção de alguma
funcionalidade do seu programa, geralmente feitos a partir de user stories
definidas durante o planejamento da sessão de desenvolvimento. Estes testes
validam os resultados esperados do sistema, do ponto de vista de um usuário.

O behave é uma biblioteca que te permite escrever seus testes em linguagem
natural (as user stories) e escrever as ações dos testes usando a linguagem
Python.


Uma aplicaçãozinha de exemplo
-----------------------------

Suponhamos que temos uma pequena aplicação web que mostra o horário o horário
atual quando acessamos a página. O código pra isso seria algo assim:

.. code-block:: python

   # -*- coding: utf-8 -*-
   # arquivo mostra_hora.py

   import datetime

   from flask import Flask


   def get_formated_datetime():
       dt = datetime.datetime.now()
       formated = dt.strftime('%d/%m/%Y %H:%M:%S')
       return formated


   app = Flask(__name__)


   @app.route('/')
   def index():
       tmpl = "<html><head><title>Que horas são?</title></head>"
       tmpl += '<body><form action="/hora"><input type="submit"" value="click!"/>'
       tmpl += '</body></html>'
       return tmpl


   @app.route('/hora')
   def show_datetime():
       tmpl = "<html><head><title>Agora são...</title></head>"
       tmpl += '<body>O horário atual é:{}</body></html>'
       dt = get_formated_datetime()
       return tmpl.format(dt)


   if __name__ == '__main__':
       app.run()

Com essa aplicaçãozinha pronta, podemos executar o seguinte comando para
iniciar a aplicação:

.. code-block:: sh

   $ python mostra_hora.py
    * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)

Agora, quando você acessar, no seu navegador, o endereço 127.0.0.1:5000
você vai ver uma página com um botão. Clicando nesse botão você cai em
outra página em que a data/hora atual são mostradas.


Os testes
---------

Agora vamos escrever os nossos testes. A primeira coisa que temos que fazer
é escrever uma user story contendo os passos que um usuário executaria no
nosso site. A história fica em um arquivo com extensão .feature, então
criaremos um arquivo chamado tests/site.feature com o seguinte conteúdo:

.. code-block:: sh

   Feature: Ver a data e hora atual no site

       Scenario: Navegar até ver o data e hora.
	   Given o usuário acessou o site
	   When ele clica no botão 'click!'
	   Then ele vê a data e hora atual.

Como vocês podem ver, este arquivo .feature descreve em linguagem natural os
passos que seriam executados por um usuário no site que deseja usar a
funcionalidade sendo executada. Vamos parar por aqui e ver em detalhes o
fizemos neste arquivo.


Given, When, Then & friends
---------------------------

As linhas com Given, When e Then são os passos que serão executados durante
o teste. A diretiva 'given' é usada para deixar o sistema em um estado
conhecido para que a partir deste estado possamos executar os testes. Nesta
diretiva você poderia criar dados, executar alguns passos ou qualquer outra
coisa que seja precisa para deixar o seu teste pronto para execução. A diretiva
'When' é usada para executar alguma ação e a diretiva 'Then' é onde verificamos
o resultado do passo executado em 'When'.

Além destas diretivas também temos as diretivas 'And' e 'But' que são diretivas
feitas somente para facilitar a escrita dos testes. Quando usarmos uma diretiva
'And' or 'But', o behave traduzirá estas diretivas como sendo a última diretiva
usada. Então, quando usamos When... And... este And é traduzido para um When
pelo behave na hora da execução dos testes.


O ambiente
----------

Em, geral, antes de começarmos os nossos testes precisamos configurar o
ambiente com algumas coisas que precisamos que já estejam rodando assim que
começarmos os testes. No caso da nossa aplicação de teste precisamos que o
servidor já esteja de pé e precisamos de uma instância de um browser
selenium para podermos simular o comportamento do usuário. Para configurar o
ambiente, o behave usa um arquivo chamado environment.py. O nosso
environment.py seria algo assim:

.. code-block:: python

   # -*- coding: utf-8 -*-

   from selenium import webdriver

   from mostra_hora import runserver, killserver


   def before_feature(context):
       """Função executada antes cada feature dos testes. O contexto
       passado aqui é compartilhado por todos os steps, então se criarmos
       algo e colocarmos como um elemento do contexto podemos recuperar o
       que criamos dentro de um step a partir do contexto."""

       # primeiro subimos o servidor
       runserver()
       # e depois criamos um browser e deixamos no contexto
       context.browser = webdriver.Chrome()


   def after_feature(context):
       """Função executada depois de cada feature dos testes."""

       # agora matamos o servidor
       killserver()
       # fechamos o browser
       context.browser.quit()


Os steps
--------

Agora que já temos nossa funcionalidade descrita no arquivo .feature precisamos
escrever um código Python que contém as ações a serem executadas durante os
testes. Os passos serão associados ao com base no texto do passo descrito no
arquivo .feature. Então vamos criar um arquivo chamado
tests/steps/site_steps.py com o seguinte conteúdo:

.. code-block:: python

   # -*- coding: utf-8f -*-

   from behave import given, when, then


   @given(u'o usuário acessou o site')
   def step_impl(context):
       """Acessamos a página inicial do site"""
       context.browser.get('http://127.0.0.1:5000')


   @when(u"ele clica no botão 'click!'")
   def step_impl(context):
       """clicamos no botão"""

       context.browser.find_element_by_tag_name('input').click()


   @then(u'ele vê a data e hora atual.')
   def step_impl(context):
       """Verificamos que a data e hora é realmente exibida."""
       browser = context.browser
       is_present = u'O horário atual é' in browser.page_source
       assert is_present

Agora já temos tudo o que precisamos para executar os testes. Isso é feito
usando o comando behave e apontando para um diretório que contenha os arquivos
.feature, assim:

.. code-block:: shell

   $ behave tests/

   Feature: Ver a data e hora atual no site    # tests/site.feature:1
       Scenario: Navegar até ver o data e hora # tests/site.feature:3
	   Given o usuário acessou o site      # tests/site.feature:4
	   When ele clica no botão 'click!'    # tests/site.feature:5
	   Then ele vê a data e hora atual.    # tests/site.feature:6

   1 features passed, 0 failed, 0 skipped
   1 scenarios passed, 0 failed, 0 skipped
   3 steps passed, 0 failed, 0 skipped, 0 undefined

E aí temos o relatório da execução dos testes indicando o que passou, o que
falhou, onde e etc.

Bom, esta foi só uma pequena introdução ao behave, para saber mais veja a
`documentação oficial <https://pythonhosted.org/behave/index.html>`_.
