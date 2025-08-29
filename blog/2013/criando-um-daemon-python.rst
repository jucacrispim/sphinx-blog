Criando um daemon com Python
============================

.. post:: Dec 03, 2013
   :tags: computisses
   :author: Juca Crispim

Fala, pessoal, tranquilidade? Como de costume, fiquei muito tempo sem postar.
Pra voltar, vamos ver como criar um daemon usando Python. Mas primeiro...


O que é um daemon?
------------------

Um daemon é um processo que fica sendo executado em segundo plano, sem contato
interativo com um usuário e desassociado de um tty. O nome daemon vem do
Demônio de Maxuel, e foi usado pela primeira vez (em computação, claro) pelo
pessoal do projeto MAC[1]. Os daemons estão presentes no Unix desde os
primórios. Aquele monte de \*d que você vê, como sshd, httpd, crond e etc, são
todos daemons.


Como criar um daemon?
---------------------

Bom, a explicação rápida pra isso é: com o bom e velho fork-off-and-die.
Cria-se um uma cópia de um processo, mata-se o processo pai e faz-se o
trabalho no processo filho. A explicação longa é a seguinte:

Cria-se um fork e sai do processo pai. Com isso, libera-se o controle de shell
se o daemon foi invocado de um. Também atribui-se outro id para o processo
filho, fazendo com que ele não seja session leader.

Criar uma nova sessão sem um terminal de controle associado.

Criar outro fork e sair novamente do pai - aquele que foi filho antes para
garantir novamente que o processo não será session leader.

Alterar a máscara de arquivos (umask).

Alterar o diretório de trabalho.

Fechar todos os descritores de arquivos descecessários.

E, agora sim executar o seu trabalho.


Por que o segundo fork()?
-------------------------

Bom, essa é uma discussão grande. Geralmente é dito que o segundo fork é
necessário para evitar que o seu daemon obtenha um terminal de controle nos
SystemV R4. Como esse já é um sistema em desuso, diz-se que o segundo fork é
desnecessário. Mas a especificação POSIX diz o seguinte[2]:

    The controlling terminal for a session is allocated by the session leader
    in an implementation-defined manner. If a session leader has no
    controlling terminal, and opens a terminal device file that is not already
    associated with a session without using the O_NOCTTY option (see open()),
    it is implementation-defined whether the terminal becomes the controlling
    terminal of the session leader

Então, como é uma questão de implementação, prefiro continuar usando um
segundo fork. :) Agora, vamos parar de falação e ir pro que importa.


O código
--------

.. code-block:: python

   #-*- coding: utf-8 -*-

   import os
   import sys
   import resource


   def create_daemon(stdout, stderr, working_dir):
       """
       cria um daemon
       """

       # faz o primeiro fork
       _fork_off_and_die()
       # cria uma nova sessão
       os.setsid()

       # faz o segundo fork
       _fork_off_and_die()

       # altera a máscara de arquivos
       os.umask(0)
       # altera o diretório de trabalho
       os.chdir(working_dir)
       # fecha todos os descritores de arquivos
       _close_file_descriptors()
       # redireciona stdout e stderr
       _redirect_file_descriptors(stdout, stderr)

   def _fork_off_and_die():
       """
       cria um fork e sai do processo pai
       """
       pid = os.fork()
       # se o pid == 0, é o processo filho
       # se o pid > é o processo pai
       if pid != 0:
	   sys.exit(0)

   def _close_file_descriptors():

       # Fechando todos os file descriptors para evitar algum
       # lock
       # RLIMIT_NOFILE é o número de descritores de arquivo que
       # um processo pode manter aberto

       limit = resource.getrlimit(resource.RLIMIT_NOFILE)[1]

       for fd in range(limit):
	   try:
	       os.close(fd)
	   except OSError:
	       pass

   def _redirect_file_descriptors(stdout, stderr):
       """
       redireciona stdout e stderr
       """

       # redirecionando stdout e stderr
       for fd in sys.stdout, sys.stderr:
	   fd.flush()

       sys.stdout= open(stdout, 'a', 1)
       sys.stderr = open(stderr, 'a', 1)

   def daemonize_func(func, stdout, stderr, working_dir, *args, **kwargs):
       """
       executa uma função como um daemon
       """
       create_daemon(stdout, stderr, working_dir)
       func(*args, **kwargs)

   class daemonize(object):
       """
       decorator para executar uma função como daemon
       """

       def __init__(self, stdout='/dev/null/', stderr='/dev/null/',
		    working_dir='.'):

	   # stdout e stderr são os lugares para onde serão redirecionados
	   # sys.stdout e sys.stderr
	   self.stdout = stdout
	   self.stderr = stderr
	   # working_dir é o diretório onde o daemon trabalhará
	   self.working_dir = working_dir

       def __call__(self, func):
	   def decorated_function(*args, **kwargs):
	       daemonize_func(func, self.stdout, self.stderr, self.working_dir,
			 *args, **kwargs)

	   return decorated_function

E agora você pode usar esse código assim:

.. code-block:: python

   #-*- coding: utf-8 -*-

   import time
   from daemonize import daemonize

   @daemonize(stdout='log_b.txt', stderr='log_b.txt')
   def b():
       print('comecando')
       time.sleep(10)
       print('fim')

   if __name__ == '__main__':

       b()

Ou assim:

.. code-block:: python

    #-*- coding: utf-8 -*-

    import time
    from daemonize import daemonize_func

    def a():
	print(time.time())
	time.sleep(50)
	print(time.time())

    if __name__ == '__main__':
	daemonize_func(a, stdout='log_a.txt', stderr='log_a.txt', working_dir='.')

É isso, pessoal. Valeu. :)

[1] http://www.takeourword.com/TOW146/page4.html

[2] http://pubs.opengroup.org/onlinepubs/007904975/basedefs/xbd_chap11.html#tag_11_01_03
