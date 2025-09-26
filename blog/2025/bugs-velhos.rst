.. post:: Sep 24, 2025
   :category: computisses
   :author: Juca Crispim


Bugs que fizeram alguns aniversários
====================================

Esses dias, numa madrugada topei com um bug no toxicbuild que existia há uns
8 anos pelo menos. No dia seguinte, de manhã no trabalho, um colega me fala
que encontrou um bug que estava lá quase há 6 anos. E além disso ser
engraçado e bem curioso que dois bugs arqueológicos tenham sido encontrados
num espaço de horas, me deixou pensando sobre a coisa de escrever *software*.


A gente até tenta
-----------------

A primeira coisa que vem à cabeça quando alguém lê isso claro que é: "porra,
mas você é burro também, hem". Errado não tá 100%, eu trabalho sozinho no
toxicbuild mesmo, poderia ser isso. Mas no trabalho somos um time, então
aí fica mais difícil culpar só a burrice. E a gente nem é desleixado,
a gente tenta fazer a coisa bonitinha. No trabalho nossas rotinas
de testes são parecidas com as do toxicbuild, que eu sei que não são perfeitas,
mas considero ser algo honesto e que se pode confiar -
até onde se pode confiar...

A primeira coisa que eu pensei foi: "preciso melhorar nossos testes" e isso é
uma verdade, sempre há espaço pra melhorar, mas as coisas nem sempre são tão
diretas assim. No caso do bug do trabalho, um teste *fuzzy* talvez tivesse
pegado, mas o nosso código lá roda em um lugar estranho com gente esquisita,
não é tão simples assim rodar uns testes doido lá. Já no caso do toxicbuild
isso não seria suficiente, o problema não era com um input estranho, era
uma interação entre o que eu escrevi e o ambiente no geral, tudo rolando
junto, tanto que só peguei o bug quando mudei as parada de servidor (e nem
foi a primeira mudança).


Mas sempre vai faltar
---------------------

Encanado com isso de melhorar as coisas duas coisas me ficaram claras: precisa
melhorar e nunca vai ser o suficiente. Nossa, descobri a pólvora agora! Todo
mundo sabe que *bug free* não existe. Mas é curioso mesmo assim depois de vinte
anos escrevendo *software* ver os bugs escapando assim.
