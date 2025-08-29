.. post:: Jan 5, 2010
   :tags: computisses
   :author: Juca Crispim

Uma pincelada no GIT
====================

O git é um programa para controle de versões. O que mais me agrada nele é a
sua flexibilidade e a sua simplicidade de uso. O que segue aí é só uma amostra
do que o git pode fazer por você. Para mais informações visite http://git-scm.com/


Apresentando-se ao git
----------------------

O que faremos aqui é versionar um projeto que já temos. Neste exemplo meu
projeto estará em /home/juca/src/novo_projeto. Então, vamos lá! Depois de
instalado o git (instruções de instalação aqui) a primeira coisa a fazer, é
apresentar-se ao git. Para tanto, vamos informar ao git nosso nome e email.

.. code-block:: sh

   $ git-config --global user.name "Juca"
   $ git-config --global user.email "juca@minhacasa.nada"


Versionando nosso primeiro projeto
----------------------------------

Agora que já nos apresentamos ao git, vamos versionar nosso projeto. A primeira
coisa é ir até o diretório onde está seu projeto, depois dar um "git init"
neste diretório para criar um novo repositório do git, e por fim, adicionar e
'commitar' os arquivos do projeto. A coisa fica assim:

.. code-block:: sh

   $ cd /home/juca/src/novo_projeto
   $ git init Initialized empty Git repository in /home/juca/src/novo_projeto/.git/
   $ git add .
   $ git commit

Com o que fizemos até agora, já temos o projeto versionado. Para ver o log dos
commits use "git log", assim:

.. code-block:: sh

   $ git log
   commit dc00b38b6beb9db50d8b0ad0fb7e6a80689c80cb
   Author: Juca
   Date: Tue Jan 5 12:41:51 2010 -0200
   Primeiro commit do tutorial

Podemos também listar os arquivos que estão sob o controle de versão usando
"git ls-files", assim:

.. code-block:: sh

   $ git ls-files arquivo1 arquivo2


Alterando e trabalhando com os arquivos do projeto
--------------------------------------------------

Agora, vamos alterar um arquivo pra ver como fica.

.. code-block:: sh

   $ emacs arquivo1

Altere o que quiser e salve o arquivo. Antes de adicionar e commitar podemos
ver o que foi alterado com "git diff":

.. code-block:: sh

   $ git diff --color arquivo1
   diff --git a/arquivo1 b/arquivo1
   index a353720..d80b33a 100644
   --- a/arquivo1 +++ b/arquivo1
   @@ -1 +1,2 @@ -Oi, eu sou o arquivo1. Essa merda é só pra brincar com o git e escrever o tuto... \ No newline at end of file +Oi, eu sou o arquivo1 (agora alterado). +Essa merda é só pra brincar com o git e escrever o tuto... \ No newline at end of file arquivo1

Obs: A opção "--color" é usada para que o resuldado do diff fique colorido, mas
aqui no tuto as cores foram pro saco... Agora, é só adicionar e commitar o
arquivo modificado. $ git add arquivo1 $ git commit


Criando branches
-----------------

Agora vamos falar de branches. Branches não custam nada no git, e são muito
fáceis de manejar. Todo novo repositório do git é criado com um branch chamado
"master". Tudo o que fizemos até agora foi neste branch. Vamos criar um novo
agora com "git branch"

.. code-block:: sh

   $ git branch # primeiro para ver os branchs que já existem
   * master

   $ git branch teste #criando o branch 'teste'
   $ git checkout teste #mudando para o branch 'teste'

Agora, vamos alterar um arquivo no branch teste.

.. code-block:: sh

   $ emacs arquivo1

Altere e salve o arquivo. Em seguida, adicione e commite o arquivo modificado

.. code-block:: sh

   $ git add arquivo1
   $ git commit

Neste ponto, temos arquivos que têm diferenças entre a versão do branch master
e do branch teste. Para "juntar" as versões dos dois branches usaremos
"git merge". O que faremos é o seguinte: Primeiro voltaremos ao branch master e
depois faremos o merge das versões. git checkout master git merge teste Agora,
depois do merge, você pode ver que as alterações feitas no branch teste também
estão visíveis no branch master. Sendo assim, podemos apagar o branch teste

.. code-block:: sh

   $ git branch -D teste


Trabalhando com os outros
-------------------------

Bom, até agora foi o basicão do "eu trabalhando comigo mesmo", mas a graça da
coisa é distribuir o código e desenvolver com os outros, não? Então! agora
vamos ver como usar o git para distribuir (ou receber de outros) o código.


Clonando um projeto
-------------------

Vamos supor que eu (Juca) estou em hostjuca e um outro cara (Zé) está em hostze
e quer contribuir com o projeto. Para fazer isto, o git pode usar ssh ou http.
Vamos começar pelo ssh é qué mais simples. O Zé precisa ter acesso liberado no
ssh da minha máquina. hostjuca será o repositório "principal" O Zé ainda não
tem uma cópia do projeto, então a primeira coisa a fazer é clonar o projeto
com "git clone". A sintaxe do "git clone" é assim (lembrando que estamos usando ssh):

.. code-block:: sh

   git clone ssh://[usuario@]host:[porta]/caminho/pro/repositorio

Então, pro Zé clonar meu projeto, a coisa ficaria assim:

.. code-block:: sh

   ze@hostze:~/src$ git clone ssh://ze@hostjuca:22/home/juca/src/novo_projeto

   Initialized empty Git repository in /home/ze/src/novo_projeto/.git/
   ze@hostjuca\'s password:

   remote: Counting objects: 7, done.
   remote: Compressing objects: 100% (6/6), done.
   remote: Total 7 (delta 0), reused 0 (delta 0)
   Receiving objects: 100% (7/7), done.

Agora o Zé já tem os arquivos do meu projeto e pode brincar a vontade.

.. code-block:: sh

   ze@hostze:~/src$ cd novo_projeto
   ze@hostze:~/src/novo_projeto$ vi arquivo2 #O Zé é filho do capeta, vocês viram, né?
   ze@hostze:~/src/novo_projeto$ git add arquivo2
   ze@hostze:~/src/novo_projeto$ git commit


Sincronizando os repositórios
-----------------------------

Agora as versões do Juca e do Zé estão diferentes. Temos duas opções para
sincronizar estas duas versões. Ou o Juca 'puxa' as mundaças do Zé ou o Zé
empurra as mudanças dele para o repositório do Juca. Antes de prosseguirmos,
vale uma explicaçãozinha sobre o comportamento do git. O git não gosta que
você empurre o mesmo branch que você clonou (ou puxou). Parece estranho, mas
tem sentido. Tente empurrar o branch master que você verá um aviso muito
explicativo.


Empurrando as coisas
--------------------

Como o Zé já tem acesso a máquina do Juca, vamos deixá-lo empurrar as mudanças
para o repositório principal. Isso será feito com "git push".
A sintaxe do "git push" é a seguinte:

.. code-block:: sh

   git push ssh://[usuario@]host:[porta]/caminho/pro/repo branch_local

Por causa do comportamento do git, o Zé tem que primeiro criar um novo branch
e empurrar este novo branch. A coisa fica assim:

.. code-block:: sh

   ze@hostze:~/src/novo_projeto$ git branch branchdoze
   ze@hostze:~/src/novo_projeto$ git push ssh://ze@hostjuca:22/home/juca/src/novo_projeto branchdoze
   ze@hostjuca\'s password:
   Counting objects: 5, done.
   Delta compression using up to 2 threads.
   Compressing objects: 100% (3/3), done.
   Writing objects: 100% (3/3), 349 bytes, done.
   Total 3 (delta 0), reused 0 (delta 0) To ssh://ze@hostjuca:22/home/juca/src/novo_projeto

   * [new branch] branchdoze -> branchdoze

Com isso, lá no repositório principal (hostjuca) foi criando um branch chamado
branchdoze, e para terminar a operação o Juca (dono do repositório principal)
precisa fazer um 'merge' do branch 'branchdoze' para o branch 'master'.

.. code-block:: sh

   juca@hostjuca:~/src/novo_projeto$ git merge branchdoze

Com isso temos o código que está em hostze e o código que está em hostjuca
(o repo principal) sincronizados.


Voltando no tempo...
--------------------

Lembra que falei que tinhamos duas opções pra sincronizar o código: empurrar ou
puxar? Então, o que fizemos até aqui foi empurrar o código de um desenvolvedor
para o repositório principal. Vamos fazer o contrário (o repo principal puxa as
alterações do dev) só pra ver como fica. Pra isso, vamos voltar o código do
repositório principal para a versão anterior as mudanças feitas pelo Zé lá em
hostze. Isso é feito com "git reset". A sintaxe é a seguinte:

.. code-block:: sh

   $ git reset --hard <commit>

Então, será feito o seguitne no repositório principal: primeiro daremos um
"git log" pra ver qual o hash do commit que queremos e depois usaremos o git
reset para voltar para esta versão. Assim:

.. code-block:: sh

   juca@hostjuca:~/src/novo_projeto$ git log
   commit 2117cfdf7a4ff0683487a33008e3fc5bca42cdbe
   Author: Zé Date: Tue Jan 5 15:49:24 2010 -0200
   Primeira alteração do Zé

   commit 937f17e96ee04e1221783e664d270dda5d87657d
   Author: Juca Date: Tue Jan 5 14:37:20 2010 -0200
   Só testando o novo branch

   commit dc00b38b6beb9db50d8b0ad0fb7e6a80689c80cb
   Author: Juca Date: Tue Jan 5 12:41:51 2010 -0200
   Primeiro commit do tutorial

   juca@hostjuca:~/src/novo_projeto$ git reset --hard 937f17e96ee04e1221783e664d270dda5d87657d
   HEAD is now at 937f17e Só testando o novo branch

Com isso temos o nosso repositório principal de volta a época em que o Zé não
tinha alterado nada.


Puxando as coisas
------------------

Agora vamos sincronizar fazendo com que o repositório principal 'puxe' as
mudanças do repositório do Zé. Isso será feito com "git pull". A sintaxe é a
seginte:

.. code-block:: sh

   $ git pull ssh://[usuario@]host:[porta] branch_remoto

Então, pro Juca puxar as mudanças do Zé, a coisa fica assim:

.. code-block:: sh

   juca@hostjuca:~/src/novo_projeto$ git pull ssh://juca@hostze:22/home/ze/src/novo_projeto master
   juca@hostze\'s password:
   From ssh://hostze:22/home/ze/src/novo_projeto
   * branch master -> FETCH_HEAD
     Updating 937f17e..2117cfd
     Fast forward arquivo2 | 3 ++- 1 files changed, 2 insertions(+), 1 deletions(-)

Como não existia nenhum conflito entre as versões, tudo ocorreu tranqüilamente, e
já temos novamente nossos códigos em hostze e hostjuca sincronizados. Bom, isso é
só uma amostra do que o git pode fazer, mas o git pode fazer muito mais que isso.
Dá uma olhadinha no gitmagic pra você ver. Na minha próxima postagem sobre o
git, explico como configurar o git para usar http ao invés de ssh. Então, até
lá e divirtam-se!
