@python - Entendendo decorators
===============================

.. post:: Dec 22, 2013
   :category: computisses
   :author: Juca Crispim


Fala, pessoal. Tranquilidade? Hoje eu vou falar sobre decorators do Python.
Os decorators são aquelas coisas, começadas por uma arroba, que você usa
"em cima" das suas definições de funções/métodos/classes. Algumas que você
já deve ter usado são:

.. code-block:: python

   class Classe:

       @classmethod
       def método_de_classe(cls):
	   print('bla')

       @property
       def read_only(self):
	   return True

Eles servem pra alterar o comportamento das nossas funções/métodos/classes
de diversas maneiras. Nos nossos exemplos aí, o primeiro decorator faz com que
o nosso método receba a classe e não a instância como primeiro parâmetro, e no
segundo, faz com que chamemos nosso método como se fosse um atributo, e se
tentarmos fazer uma atribuição a isso, vai dar erro.

Deu pra perceber que dá pra fazer um monte de coisa legal com os decorators,
não? E pra usar já vimos também que não é difícil, é só colocar um
"@meu_decorator" em cima das nossas declarações, e já era.

Pra escrever também não é difícil, a gente só precisa entender direitinho o
que é um decorator e depois fica fácil. Então, vamos lá!


O que é um decorator?
---------------------

Bom, a definição mais simples de um decorator é a seguinte: Um decorator é um
callable que retorna um callable. Ponto. Callable é um cara que você pode
"chamar" - tipo coisa() - como uma função, um método...  qualquer objeto que
tenha o método __call__.

Seguindo por essa linha, podemos fazer assim, o nosso primeiro decorator:

.. code-block:: python

   #-*- coding: utf-8 -*-

   def um_decorator(func):
       """
       Um callable que tem um callable como parâmetro
       e retorna um callable.
       """
       # faz nada...
       print('decorating func')
       return func

E, no shell, usamos assim:

.. code-block:: sh

   >>> @um_decorator
   ... def some_func():
   ...     print('oi')
   ...
   decorating func
   >>> some_func()
   oi
   >>>

Vamos parar por aqui e entender o que aconteceu.


Entendendo o funcionamento do decorator
---------------------------------------



Assim que definimos nossa função, o interpretador 'viu' que esta função estava
decorada, isto é, havia algo começando por um arroba antes da definição e
adicionou às variáveis globais, usando o nome da sua fução, não a função que
você definiu, mas o retorno do seu decorator. Isso significa que assim que o
interpretador 'viu' que sua função estava decorada, ele fez algo que seria tipo
isso:

.. code-block:: sh

   >>> some_func = um_decorator(some_func)
   decorating func
   >>>

E aqui que está toda a jogada. Agora, quem está usando o nome 'some_func' não é
mais a função que você definiu, e sim a função que o nosso decorator retornou.

Pra exemplificar melhor, vamos fazer uma segunda versão do decorator:

.. code-block:: python

   #-*- coding: utf-8 -*-

   def um_decorator(func):
       """
       Um callable que tem um callable como parâmetro
       e retorna um callable.
       """
       def other_func():
	   print('ola')

       return other_func

E no shell fica assim:

.. code-block:: sh

   >>> @um_decorator
   ... def some_func():
   ...     print('oi')
   ...
   >>> some_func
   <function um_decorator.<locals>.other_func at 0x7fac11ffe050>
   >>> some_func()
   ola
   >>>

Então, o que acabamos de ver aí é que quem está usando o nome 'some_func' não
é a função que definimos e sim a função other_func, que definimos dentro do
nosso decorator. Legal, né?


O que fizemos até aqui foi inútil, eu sei, mas vamos melhorar daqui pra
frente. Prometo. :)


Um decorator melhorzinho
------------------------

O que a gente vai fazer agora é o seguinte:  um decorator pra logar  as coisas
antes e depois da execução de alguma coisa. Algo mais ou menos assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   def loga(func):
       """
       Decorator que loga a execução do callable
       """
       def loga_execucao(*args, **kwargs):
           print('iniciando execucao com %s, %s' % (str(args), str(kwargs)))
           retorno = func(*args, **kwargs)
           print('terminou execucao com %s' % retorno)

       return loga_execucao

E, novamente, no shell fica assim:

.. code-block:: sh

   >>> @loga
   ... def some(a, b):
   ...     return a + b
   ...
   >>> some(1, 1)
   iniciando execucao com (1, 1), {}
   terminou execucao com 2
   >>> from random import random
   >>> @loga
   ... def do_magic(*args, **kwargs):
   ...     return random()
   ...
   >>> do_magic(1, 'asdf', nada='não sei', acre=NotImplemented)
   iniciando execucao com (1, 'asdf'), {'nada': 'não sei', 'acre': NotImplemented}
   terminou execucao com 0.28372230130165577
   >>>

O funcionamento básico aqui é a mesma coisa do nosso outro decorator, a função
loga retornou a função loga_execucao e esta função está usando o nome da
função que foi decorada. Mas, além disso, tiveram umas coisas um pouco
diferentes, então vamos parar por aqui, respirar um pouco e ver tudo com calma.


Entendendo o funcionamento do decorator melhorzinho
---------------------------------------------------

A primeira coisa diferente que notamos agora é que a função loga_execucao tem
como parâmetros \*args e \*\*kwargs (linha 8). Isso significa que vale tudo,
aceita quaisquer argumentos - Nota à parte: isso, do \*args e \*\*kwargs, é
MUITO da hora. Isso porque queremos decorar qualquer coisa e qualquer coisa
pode ter qualquer argumento.

A outra coisa diferente é o que importa aqui. Na linha 10, a gente chama a
func(), que é a função que passamos para o decorator. Apesar de a função
'func ' não estar no namespace da função loga_execucao (linhas 8-11), a quando
chamamos func(), ela é recuperada do namespace antecessor, isto é, da função
loga (linhas 4-12). Então, quando a toda vez que a função loga_execucao for
executada, o interpretador vai lembrar que no momento da criação dela, existia
um parâmetro chamado func, que é a função que você passou pro decorator.

E é isso, essa é toda a mágica dos decorators. Aqui você já pode fazer muitas
coisas legais com eles, mas ainda tem mais!


Um decorator com classe
-----------------------

Bom, agora nós vamos fazer um decorator usando uma classe, não funções. O
esquema de funcionamento é o mesmo, um callable que retorna outro callable.
O nosso decorator agora será um decorator para cachear funções custosas. Se uma
fução demora muito pra executar, deixamos o resultado em memória e da próxima
vez já pegamos o resultado computado. O decorator fica mais ou menos assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   CACHE = {}


   class cacheado:
       def __init__(self, func):
	   self.func = func

       def __call__(self, *args, **kwargs):
	   # cacheia a função
	   name = self.func.__name__
	   cacheado = CACHE.get(name)
	   if not cacheado:
	       cacheado = self.func(*args, **kwargs)
	       CACHE[name] = cacheado

	   return cacheado

E no shell fica assim:

.. code-block:: sh

   >>> @cacheado
   ... def take_time():
   ...     lista = []
   ...     for i in range(100000):
   ...         lista.insert(0, i)
   ...     return list
   ...
   >>> take_time
   <__main__.cacheado object at 0x7f8c1fd81790>
   >>> type(take_time)
   <class '__main__.cacheado'>
   >>>
   >>> timeit.timeit(take_time, number=1)
   4.054789036999864
   >>> timeit.timeit(take_time, number=1)
   9.35900243348442e-06
   >>> timeit.timeit(take_time.__call__, number=1)
   1.2202999641885981e-05
   >>>

Viram? Na primeira vez levou 4 segundos. Da segunda foi instantâneo. Agora
vamos entender direito o que aconteceu aí.


Entendendo o decorator com classe
---------------------------------

Quando fazemos @cacheado, estamos fazendo algo assim, lembra?

.. code-block:: sh

   >>> def other_take_time():
   ...     sleep(10)
   ...     return True
   ...
   >>> other_take_time = cacheado(other_take_time)
   >>> other_take_time
   <__main__.cacheado object at 0x7f8c1eca0690>
   >>>

Então, como agora nosso decorator é uma classe, a função decorada é uma
instância da classe 'cacheado', e quando chamamos a função decorada, estamos
na verdade chamando o método __call__ da instância de 'cacheado'. Simples
também, não?


Um decorator com parâmetros
---------------------------

Vamos fazer um outro decorator 'cacheado', mas agora aceitará como argumento
quantos segundos o resultado ficará cacheado. Assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   from time import time

   CACHE = {}


   class cacheado:

       def __init__(self, tempo):
	   """
	   Recebe o tempo, em segundos, que o resultado
	   da função ficará cacheado.
	   """
	   self.tempo = tempo

       def __call__(self, func):
	   # cacheia a função
	   # agora, __call__ será chamado pelo @ na construção
	   # da função, isto é, uma vez só, e sendo assim, __call__
	   # tem que retornar um callable. Agora, __call__
	   # é o nosso decorator.

	   def cacheia_resultado(*args, **kwargs):
	       name = func.__name__
	       agora = time()
	       cacheado = CACHE.get(name)
	       if not cacheado or ((cacheado[1] + self.tempo) < agora):
		   retorno = func(*args, **kwargs)
		   cacheado = (retorno, agora)
		   CACHE[name] = cacheado
	       retorno = cacheado[0]

	       return retorno

	   return cacheia_resultado

E usamos assim:

.. code-block:: sh

   >>> @cacheado(10)
   ... def take_time():
   ...     lista = []
   ...     for i in range(100000):
   ...         lista.insert(0, i)
   ...     return lista
   ...
   >>> take_time
   <function cacheado.__call__.<locals>.cacheia_resultado at 0x7f8c1ec9f680>
   >>> timeit.timeit(take_time, number=1)
   4.03598285499902
   >>> timeit.timeit(take_time, number=1)
   0.015197464999801014
   >>> # alguns segundos depois
   ...
   >>> timeit.timeit(take_time, number=1)
   4.0627139449934475

A coisa aí mudou um pouco de figura agora, vamos parar de novo pra entender.


Entendendo o decorator com parâmetro
------------------------------------

Agora, quando fizemos @cacheado(10), fizemos algo assim:

.. code-block:: sh

   >>> def other_take_time():
   ...     sleep(10)
   ...     return True
   ...
   >>> decorator = cacheado(10)
   >>> decorator
   <__main__.cacheado object at 0x7f8c1eca0590>
   >>> other_take_time = decorator(other_take_time)
   >>> other_take_time
   <function cacheado.__call__.<locals>.cacheia_resultado at 0x7f8c1ec9fd40>
   >>>

Aí, o decorator não éra mais uma classe ou uma função, e sim uma instância da
classe 'cacheado'. Com isso, o método __call__ é chamado na criação da função
decorada, e não é mais a função decorada, como no exemplo anterior. A função
decorada agora é 'cacheia_resultado', a função que definimos dentro do método
__call__. Tricky, mas simples, não? Pouco código pra uma coisa legal dessas...
Da hora.

Estamos quase lá, mas vamos fazer mais um, só por curiosidade...


Um decorator com parâmetro opcional
-----------------------------------

Agora que a gente já sacou como funcionam os decorators, fica mole fazer um
com parâmetro opcional. Aposto que você já tá pensando em como se faz. Então
vamos escrever logo isso.

.. code-block:: python

   #-*- coding: utf-8 -*-

   from time import time

   CACHE = {}


   class _cacheado:

       def __init__(self, tempo):
	   """
	   Recebe o tempo, em segundos, que o resultado
	   da função ficará cacheado.
	   """
	   self.tempo = tempo

       def __call__(self, func):
	   # cacheia a função
	   # agora, __call__ será chamado pelo @ na construção
	   # da função, isto é, uma vez só, e sendo assim, __call__
	   # tem que retornar um callable. Agora, __call__
	   # é o nosso decorator.

	   def cacheia_resultado(*args, **kwargs):
	       name = func.__name__
	       agora = time()
	       cacheado = CACHE.get(name)

	       if not cacheado or ((cacheado[1] + self.tempo) < agora):
		   retorno = func(*args, **kwargs)
		   cacheado = (retorno, agora)
		   CACHE[name] = cacheado

	       retorno = cacheado[0]
	       return retorno

	   return cacheia_resultado


   def cacheado(param):
       """
       param pode ser tanto um callable - no caso de o decorator
       não ser usado com parâmetro ou o tempo, se usado com parâmetro.
       """

       if not callable(param):
	   # usou o decorator com parâmetro, assim:
	   # cacheado(5)
	   decorator = _cacheado(param)
	   return decorator

       else:
	   # usado sem parâmetro, usaremos o tempo padrão.
	   tempo = 10
	   decorator = _cacheado(tempo)
	   # Ao invés de retornar o decorator, como quando usando
	   # com parâmetro, temos que retornar a função decorada
	   # pela instância de _cacheado por que a funçaõ 'cacheada'
	   # já foi usada como o decorator
	   função_decorada = decorator(param)
	   return função_decorada

E usamos assim:

.. code-block:: sh

    >>> @cacheado
    ... def take_time():
    ...     lista = []
    ...     for i in range(100000):
    ...         lista.insert(0, i)
    ...
    >>> timeit.timeit(take_time, number=1)
    4.068460593000054
    >>> timeit.timeit(take_time, number=1)
    1.612800406292081e-05
    >>>
    >>> @cacheado(30)
    ... def other_take_time():
    ...     sleep(10)
    ...     return True
    ...
    >>> timeit.timeit(other_take_time, number=1)
    10.007121031994757
    >>> timeit.timeit(other_take_time, number=1)
    1.7487000150140375e-05
    >>>

Belezinha, né? Acho que chegamos inteiros ao fim e deu pra sacar que não tem
nada de mistério com os decorators, bem ao contrário, certo?

Ficaram dúvidas? Pode perguntar! :)

Hum... ficou grande, né? Será que alguém leu até aqui?

.. code-block:: sh

   [juca@debianmental:~/mysrc/exemplos/decorator]$ python3
   Python 3.3.3 (default, Nov 27 2013, 17:12:35)
   [GCC 4.8.2] on linux
   Type "help", "copyright", "credits" or "license" for more information.
   >>> from decorator import black_knight
   >>> @black_knight
   ... def multiplica(a, b):
   ...     return a*b
   ...
   >>> multiplica(2, 3)
   None shall pass
   >>>
   [juca@debianmental:~/mysrc/exemplos/decorator]$ su
   Senha:
   root@debianmental:/home/juca/mysrc/exemplos/decorator# python3
   Python 3.3.3 (default, Nov 27 2013, 17:12:35)
   [GCC 4.8.2] on linux
   Type "help", "copyright", "credits" or "license" for more information.
   >>> from decorator import black_knight
   >>> @black_knight
   ... def multiplica(a, b):
   ...     return a*b
   ...
   >>> multiplica(2, 3)
   You are a looney.
   6
   >>>

Kudos pra quem postar o código de 'black_knight'!
