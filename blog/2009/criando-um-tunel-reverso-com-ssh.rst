.. post:: Dec 22, 2009
   :category: computisses
   :author: Juca Crispim


Criando um túnel reverso com SSH
================================

Essa aqui é uma rapidinha sobre como criar um túnel reverso no ssh.
Pergunta: Pra que serve um túnel ssh reverso? Resposta: Eu uso isso pra chegar
em uma máquina que não poderia chegar diretamente (máquina atrás de nat
e talz...) Vamos lá!

Suponhamos que eu tenha dois hosts distintos (host1 e host2). host1 está atrás
do nat e host2 está conectado diretamente na internet. O que faremos aqui é
abrir uma conexão ssh de host1 para host2 e deixar o túnel aberto, assim
podendo ir de host2 para host1 (adeus nat!). A sintaxe para sair de host1 para
host2, deixando o túnel aberto, seria assim:

.. code-block:: sh

   ssh -R [porta_pra_voltar_pra_host1]:localhost:[porta_ssh_host2] host2

Então, eu estando em host1 faria o seguinte:

.. code-block:: sh

   juca@host1:~$ ssh -R 2222:localhost:22 host2

Ai, enquanto (e somente enquanto) esta conexão estiver aberta, é possivel
conectar de host2 em host1 usando: ssh -p [porta_pra_voltar_pra_host1]
localhost Então, eu estando em host2, faria o seguite:

.. code-block:: sh

   juca@host2:~$ ssh -p 2222 localhost Bom... É isso ai. Molezinha, não?
