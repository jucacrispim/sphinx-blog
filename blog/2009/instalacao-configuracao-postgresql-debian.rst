.. post:: Dec 21, 2009
   :category: computisses
   :author: Juca Crispim

Instalação e configuração do PostgreSQL no Debian
=================================================

Essa aqui é uma rapidinha sobre como instalar e configurar o PostgreSQL no
Debian.

Primeiro, instalar o banco:

.. code-block:: sh

   # aptitude install postgresql

Depois de baixar e instalar é hora de configurar. O usário root do nosso banco
de dados é o postgres. No processo de instalação foi criado um usuário chamdo
postgres também no sistema. Então, nos logaremos com este usuário.

.. code-block:: sh

   # su postgres

Por padrão, o usuário de banco de dados 'postgres' não tem senha, então agora
nos logaremos no shell do PostgreSQL para alterar a senha do usuário postgres.

Primeiro, logando no shell...

.. code-block:: sh

   $ psql

Agora, já no shell do PostgreSQL, vamos alterar a senha do usuário postgres

.. code-block:: sh

   postgres=# ALTER USER postgres WITH PASSWORD 'qualquersenha';

Esse cara que a gente acabou de configurar ai é o root do banco de dados...
A gente não vai ficar usando esse usuário nas nossas aplicações, né?
Então! vamos criar um novo usuário.

.. code-block:: sh

   postgres=# CREATE USER usuario NOCREATEDB NOSUPERUSER NOCREATEROLE PASSWORD 'senha';

Agora, vamos criar uma tabela também

.. code-block:: sh

   postgres=# CREATE DATABASE minhabase;

Bom, já criamos usário, base de dados... Agora precisamos configurar o modo
como os clientes se autenticarão no servidor. Essas configurações se encontram
no arquivo pg_hba.conf, que no Debian fica em /etc/postgresql/8.4/main.

OBS: Esse 8.4 aí em cima se refere à versão que estou usando. Se sua versão for
diferente, o número também será. Vamos abrir o arquivo e editá-lo.

Como o arquivo é bem comentado não vou me alongar na explicação...
Qualquer coisa é só olhar aqui. A primeira coisa que farei aqui é deixar o tipo
de autenticação para os usuários locais como md5.

Então, a linha que era assim: `local all all ident` Ficou assim:
`local all all md5`

Do jeito que as coisas estão, somente usuários locais poderão se conectar ao
banco. Então agora vamos fazer umas configurações para que clientes remotos
possam se conectar ao banco. O primeiro passo para liberar conexões para
clientes remotos é adicionar uma liniha ao pg_hba.conf. A linha seria algo
como: `host all all 0.0.0.0/0 md5`.

Pra finalizar precisaremos também editar o arquivo postgresql.conf, que fica no
mesmo diretório que o pg_hba.conf.

No postgresql.conf procure pela linha `#listen_addresses = 'localhost'` e mude
para `listen_addresses = '*'`

Agora é só reiniciar o postgresql...

.. code-block:: sh

   /etc/init.d/postgresql-8.4 restart

E está tudo pronto!
