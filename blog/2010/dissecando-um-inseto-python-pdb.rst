.. post:: Aug 16, 2010
   :tags: computisses
   :author: Juca Crispim


Dissecando um inseto (ou como usar o pdb, the Python Debugger)
==============================================================

Boas, pessoal! Hoje eu estou aqui pra mostrar como se usa o (básico do) Python
Debugger, o pdb, que é (óbvio) um depurador pra se usar com o Python. Este
exemplo é feito usando o Python 3.


Mas... como é?
--------------

Bom, existe mais de uma maneira de usar o pdb. A que vou mostrar é chamando o
pdb como um script para depurar outro script. Então, primeiro, vamos criar um
scriptosco com um erro mais tosco ainda pra gente começar a brincar com o pdb.
O scriptosco é o seguinte:

.. code-block:: python

   #-*- coding: utf-8 -*-

   def divide(a, b):
       erro = False
       try:
	   return float(a) / float(b)
       except:
	   erro = True

       if erro:
	   raise Exception("Não vou te dizer qual é o erro...")

   a = 1
   b = 2
   divide(a, b)
   a = 3
   b = 0
   divide(a, b)

Bom, rodando isso aí, a gente tem o seguinte:

.. code-block:: sh

   $ $python3 exemplopdb.py
   Traceback (most recent call last):
     File "exemplopdb.py", line 18, in
       divide(a, b)
     File "exemplopdb.py", line 11, in divide
       raise Exception("Não vou te dizer qual é o erro...")
   Exception: Não vou te dizer qual é o erro...


Será que esse tal de pdb funciona mesmo?
----------------------------------------

Agora que temos um erro, podemos usar o pdb pra achar cara. A gente vai fingir
que não vê nada errado até a hora certa, tá? :P A sintexe pra se chamar o pdb
é a seguinte: python -m pdb <meu_script> Então, ao chamar nosso doente junto
com o pdb, a gente vai cair no shell do pdb. Assim:

.. code-block:: sh

   $ python3 -m pdb exemplopdb.py
   --Return--
   > /media/5511fc83-ad85-484b-9b0e-0948abcb6026_/virtualpython/lib/python3.1/encodings/__init__.py(67)normalize_encoding()->'utf_32_be'
   -> return ''.join(chars)
   (Pdb)

Nós começaremos criando breakpoints nas linhas 11 e 18. Fazemos isto com o
comando b[reak]. Depois, utilizando o comando c[ontinue], continuaremos a
execução do programa até que algum breakpoint seja encontrado. E por fim,
usaremos o comando list para listar um trecho de código e ver onde paramos.

.. code-block:: sh

   (Pdb) break exemplopdb.py:11
   Breakpoint 1 at /media/sda6/Projetos & afins/scripts/exemplopdb.py:11
   (Pdb) b exemplopdb.py:18
   Breakpoint 2 at /media/sda6/Projetos & afins/scripts/exemplopdb.py:18
   (Pdb) c
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(18)()
   -> divide(a, b)
   (Pdb) list
    13  	a = 1
    14  	b = 2
    15  	divide(a, b)
    16  	a = 3
    17  	b = 0
    18 B->	divide(a, b)
    19
   [EOF]
   (Pdb)

Nosso primeiro breakpoint é bem na chamada da função, então vamos "entrar na
função" e acompanhar a execução. A gente faz isso com o comando s[tep]. Uma vez
na função, a gente pode acompanhar a execução linha-por-linha usando o comando
n[ext] e também pode imprimir os valores das variáveis com print.

.. code-block:: sh

   (Pdb) s
   --Call--
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(3)divide()
   -> def divide(a, b):
   (Pdb) list
     1  	#-*- coding: utf-8 -*-
     2
     3  ->	def divide(a, b):
     4  	    erro = False
     5  	    try:
     6  	        return float(a) / float(b)
     7  	    except:
     8  	        erro = True
     9
    10  	    if erro:
    11 B	        raise Exception("Não vou te dizer qual é o erro...")
   (Pdb) n
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(4)divide()
   -> erro = False
   (Pdb) n
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(5)divide()
   -> try:
   (Pdb) n
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(6)divide()
   -> return float(a) / float(b)
   (Pdb) print(a, b)
   (3, 0)
   (Pdb) n
   ZeroDivisionError: 'float division'
   > /media/sda6/Projetos & afins/scripts/exemplopdb.py(6)divide()
   -> return float(a) / float(b)
   (Pdb)

Bom, aí matamos o erro, não? Funciona mesmo! Moral da história: As vezes o pdb
pode salvar várias horas do seu dia. Faça dele um amigo. :)
