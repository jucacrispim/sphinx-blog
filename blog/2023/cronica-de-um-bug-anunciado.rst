Crônica de um bug anunciado
---------------------------

.. post:: Apr 2, 2023
   :category: computisses
   :author: Juca Crispim


Este texto eu tinha começado a escrever em 2015. Agora voltando o blog,
pegando os backups achei e acho que tá bom de terminar.

O contexto da história
----------------------

O software livre, pouco mais de uma década depois do seu 'surgimento formal'
teve seu racha que continua até hoje. Por um lado um grupo que de denominava de
software livre e outro que se denominava como código aberto. Estes enfatizavam
a superioridade técnica sobre o modelo de desenvolvimento fechado enquanto
aqueles enfatizavam as liberdades advindas do compartilhamento de código, mas
que apesar das diferenças acabavam convivendo bem. As coisas começaram a
degringolar por volta de 2007, com as GPL v3 wars que culminaram com um
diminuição do uso da GPL e com as licenças mais permissivas tendo um destaque
maior (muitos projetos agora colocam uma licença permissiva como uma feature do
software).

Assim chegamos a um ponto onde o pessoal do software livre se sente como tendo
sido derrotado em certo sentido porque apesar do uso de software livre ter sido
disseminado enormemente ao ponto de hoje em dia o software livre ser parte do
mainstream do desenvolvimento de software nos dias de hoje, o que acabou se
disseminando na verdade foi o open source, para ser mais preciso, a ideologia
open source, já que o software é basicamente o mesmo.


Como começou?
-------------

A formação do movimento do software livre sempre se deu mais ou menos nas
seguintes premissas: Um grupo (ou um indivíduo só) de hackers desenvolvia algum
programa, e principalmente dos anos 90 em diante, colaborando via internet,
fazendo com que o desenvolvimento de software livre pudesse crescer de maneira
exponencial. O principal elo entre estes hackers era, obviamente, o software
sendo produzido, mas além disso também uma ideologia compartilhamento de
conhecimento, de bens imateriais no geral e uma certa apologia à liberdade,
seja lá o que isso for. E ao mesmo tempo em que crescia o software livre,
cresciam também as oportunidades de negócio e os contatos com as grandes
empresas de software proprietário.

A questão de modelos de negócios já existia no movimento de software livre,
com Richard Stallman falando sobre isso no manifesto gnu e pensando nos modelos
de negócio propostos no manifesto gnu podemos começar a entender o porquê da
primeria disputa entre capitalistas envolvendo o software livre: de um lado as
nascentes empresas de internet e de outro as empresas de software proprietário.
Entenda-se disputa aqui de uma maneira bem ampla, não sendo exatamente uma
disputa, mas sim uma diferença de relacionamento com o software livre ditadas
por questões econômicas. As empresas de internet desde o seu nascimento usaram
de software livre vendo aí uma alternativa viável, melhor e mais barata aos
softwares proprietários (linux + apache foi o primeiro "sucesso" do linux).

Do lado da comunidade de software livre, ao mesmo tempo, também começaram a se
ressaltar as primeiras divergências, agora entre o pessoal mais 'apolítico',
mais preocupado com software e o pessoal mais 'ideológico', preocupado com
software, mas sempre ressaltando (basicamente) a questão do usuário como o
controlador do seu ambiente computacional. O primeiro grupo com o passar do
tempo começou a se chamar de open source  ao invés de free software. A
princípio essa divisão era apresentada - principalmente pelos propomentes do
open source - como uma diferença menor,  uma diferença no modo de se lidar com
'os gravatas', mas no fim era tudo a mesma coisa, todos usavam e escreviam os
mesmos programas, usavam as mesmas licensas, eram todos parte da mesma coisa.


O que pegou?
------------

As coisas continuaram mais ou menos assim até a metade/final dos anos 2000 até
que foi lançada a GPLv3. O objetivo dessa nova versão da GPL é tapar os buracos
na redação da gpl v2 e que deixavam brechas para usos proprietários de
programas sob a gpl. Mas com a chegada da gpl v3 as diferenças entre open
source e software livre se tornaram mais visíveis e estas diferenças acabaram
sendo a expressão mais visível de um movimento para tirar da sala qualquer
coisa que pudesse trazer pruridos numa reunião com executivos. Nessa época as
licensas mais permissivas começaram a ser mais usadas, a gpl perdeu relevância
e o software livre virou 'oficialmente' open source. Curiosamente, em certo
sentido, as 'previsões' de ambos, software livre e open source se
concretizaram.

Com o avanço das licensas permissivas, as empresas perderam o 'medo de
contaminação' com a gpl e software open source se tornou cada vez mais presente
até ser padrão em muitos lugares. Mais do que isso, software livre é a espinha
dorsal da internet, rodando em servidores e roteadores e atualmente também a
base de muito software desenvolvido em cima de bibliotecas livres.


Oito anos depois
----------------

Foi aqui que parei de escrever em 2015 e depois de oito anos a marcha dos
acontecimentos continuou acelerada no caminho que estava. Software livre faz
parte do mainstream do desenvolvimento de software corporativo. Em qualquer
projeto de software hoje em dia você vai encontrar muito software livre ou no
software em si ou na tool chain, em algum lugar você vai encontrar algum
código livre. O principal dispositivo computacional das pessoas hoje em dia é
um telefone. A maioria dos telefones roda android e android é feito em cima de
um monte de coisa livre (mas não gpl porque o google tem medo de infecção). Mas
ao mesmo tempo o ambiente computacional no telefone é uma das coisas mais
travadas que há além de estar cheio de aplicativos proprietários. Como?

No ponto de se ter um monte de coisas proprietárias feitas em cima de software
livre é o resultado da proliferação de licensas sem copyleft. Licensas com
copyleft tem uma caracteristica principal que é fazer com que todos participem
do jogo. "Tá aqui meu código, pra participar da brincadeira você também
contribui com código". Já licensas sem copyleft simplesmente dão as 4
liberdades, mas não demandam código em troca. E quando você dá um monte de
coisas pra um negócio sem demandar nada em troca, não vai haver troca. Negócios
são negócios.

Por outro lado a maioria das pessoas usa seus dispositivos geralmente como um
terminal burro, toda a computação é feita em servidores e os dispositivos
mostram o resultado e se nem a gpl v3 ganhou tração imagina a agpl que foi
feita pra lidar exatamente com servidores. Então a gente chegou ao ponto onde
nunca se teve tanto software livre sendo feito e usado e ao mesmo tempo o
ambiente computacional das pessoas nunca foi tão travado.


Por que o bug era anunciado?
----------------------------

O movimento de software livre causou movimentos muito fortes e causou avanços
incríveis na indústria de software. Nos anos 80 um compilador de c custava
alguns milhares de dólares, nos anos 90 todo ambiente de desenvolvimento era
com software proprietário e hoje as coisas são bem diferentes. Mas apesar disso
as empresas de tecnologia estão no controle da informação e da computação
com maior preponderância ainda. Aconteceram flame wars de todo tipo, posts
gigantescos pseudo-filosóficos sobre o que é liberdade e no final das contas o
que importava mesmo pros negócios é o quanto de trabalho grátis elas conseguem
extrair.

O caso do software livre foi um caso de se olhar pras árvores e não ver a
floresta. Enquanto mirávamos em como fazer nossos programas melhores e mais
úteis, a gente esqueceu que programas de computador são só ferramentas e
ferramentas são sempre usadas por pessoas dentro de um contexto. O nosso
contexto é que a produção de qualquer coisa (inclusive de programas de
computador) que sirva para atender alguma necessidade humana está submetida à
logica da maximização e apropriação de lucros.

Por esta lógica, ter o controle dos dados, da computação, dos algoritmos
usados por todos e cada vez mais importatantes é extremamente lucrativo e
enquanto estivermos submetidos a este contexto, as nossas licensas de software
podem ganhar muitas batalhas, mas a guerra vai sempre ser perdida.
