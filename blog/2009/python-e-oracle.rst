.. post:: Nov 26, 2009
   :category: computisses
   :author: Juca Crispim

Python + Oracle
===============

Aqui no trampo o pessoal usa banco de dados Oracle. E sempre preciso fazer
alguns scripts em Python pra trabalhar com o Oracle. Então aqui vai uma
rapidinha sobre conectar no Oracle usando Python.

Primeiro você precisa instalar o oracle client
(instalação fora do escopo do post... STFW). Depois do client instalado, é
preciso instalar o módulo cx_Oracle (está aqui), para conectar no banco através
do Python.

Depois do módulo instalado a coisa fica fácil fácil.

.. code-block:: sh

   >>> import cx_Oracle
   >>> ora_conn = cx_Oracle.connect(user, passwd, host/service_name)
   >>> ora_cur = ora_conn.cursor()
   >>> ora_cur.execute("SELECT * FROM DUAL")
   >>> resultado = ora_cur.fechall()
   >>> ora_conn.close()

Fácil, né?

Resolvendo problema
-------------------

Se na hora do import ocorrer um erro assim:

.. code-block:: sh

   >>> import cx_Oracle
   Traceback (most recent call last): File "", line 1, in File "build/bdist.linux-i686/egg/cx_Oracle.py", line 7, in File "build/bdist.linux-i686/egg/cx_Oracle.py", line 6, in __bootstrap__ ImportError: libclntsh.so.10.1: cannot open shared object file: No such file or directory
   >>>

O problema é que o oracle não está no PATH do seu sistema. Para resolver isto,
no bash, eu uso:

.. code-block:: sh

   $ export ORACLE_HOME=/usr/lib/oracle/xe/app/oracle/product/10.2.0/server/
   $ export LD_LIBRARY_PATH=$ORACLE_HOME/lib
   $ export PATH=$ORACLE_HOME/bin:$PATH

Bom, é isso aí... Mamão-com-açucar!
