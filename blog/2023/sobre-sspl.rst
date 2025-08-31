Sobre a SSPL
============

.. post:: May 22, 2023
   :category: computisses
   :author: Juca Crispim


Bom, eu sei que tô atrasado, mas eu sou lerdo mesmo, então vai.

A SSPL (Server Side Public Licence) é a licença criada pelo pessoal do mongodb
para ser usada no lugar da AGPLv3 que era a licença do mongodb. A SSPL é
basicamente a AGPLv3 com a cláusula 13 modificada. Esta cláusula diz respeito
a como a licença se relaciona com a execução remota do programa licenciado.

A AGPL diz em sua cláusula 13 que um programa modificado, mesmo que seja
acessado via rede, tem que ter o código fonte disponível e que um programa
licenciado pela agpl pode ser combinado tranquilamente com um programa
licenciado sob gpl. A SSPL por outro lado, além de exigir que um programa
modificado tenha o código disponível assim como a agpl, também diz que se for
oferecido um serviço de hospedagem do programa, todo o código de suporte ao
serviço tenha que ser licenciado sob sspl de maneira que um usuário possa
hospedar seu próprio serviço.


Por que mais uma licença?
-------------------------

A SSPL existe para lidar com as grandes de hospedagem (aws, microsoft e google)
que oferecem serviços em cima do mongo e estão  comendo uma fatiga gigante
do mercado da mongo inc. tanto que a amazon anunciou um fork do mongodb assim
que ele passou a ser licenciado com a sspl. A ideia da alteração feita na agpl
é de  forçar os cloud providers a abrirem sua infra, ou mais realisticamente,
pagarem a licença pra mongo inc. E aqui nem vou entrar no mérito da briga de
duas empresas multi-milionárias, a questão é mais a licença do software mesmo.

A licença foi submetida para analise da OSI e acabou não sendo aprovada, com
basicamente o principal argumento que a licença é restritiva do uso do software
assim violando o princípio de que o software deve poder ser usado da maneira que
se quiser.


Restritiva como?
----------------

Um dos pontos do software livre é a não discriminação por uso do software.
O exemplo clássico é o do aborto: um software livre não pode restringir o uso
nem a uma clinica de aborto e  nem a uma igreja anti-aborto e um dos argumentos
era de que a licença restringia o uso baseado no tipo de uso, restringindo o
uso baseado em ser um serviço ou não. E é aqui que eu acho estranho o argumento
da osi.

Toda licença com copyleft tem alguma restrição na criação de trabalhos
derivados, em geral exigindo que trabalhos derivados sejam lançados sobre a
mesma licença e sendo assim "restringe" a liberdade de software proprietário,
mas é uma restrição que faz bem ao ecossistema de software livre. No meu modo
de ver a sspl para lidar com uma situação que não era contemplada com as
licenças anteriores, isto é, um mundo com cloud providers gigantescos, como
isto afeta a distribuição (ou falta de) dos programas. Me pareceu que o
argumento da osi é o mesmo argumento que se usa contra licenças de copyleft.


A coisa do cachimbo e da boca torta
-----------------------------------

Como comentei no outro post, tem um tempo que as licenças com copyleft caíram
em desuso e estão sempre 'sob ataque', e o open source é um dos responsáveis
por isso. Quando do lançamento da GPLv3, que veio pra 'tapar buracos' da GPLv2,
um pessoal torceu o nariz. O pessoal não tinha como dizer que uma licença da
fsf não é livre, mas fizeram questão de promover licenças mais amigávies aos
negócios. E depois de tanto ser amigáveis aos negócios agora a osi passou a
tomar lado na briga entre empresas.

No meu modo de ver, a sspl apesar de ser uma arma de negócios, é uma licença
que adere aos princípios do software livre e expande isso, fazendo com que
cloud providers também tenham que participar do jogo. Cinco anos já se passaram,
o código do mongo continua aí disponível, pode-se usar o mongo da maneira que quiser,
e se quiser montar um serviço de hospegagem de mongo também pode, só precisa abrir
o seu código usado na infra. Acho justo e no espírito das licenças com copyleft.
