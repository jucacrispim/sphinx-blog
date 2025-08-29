.. post:: Aug 17, 2025
   :tags: computisses
   :author: Juca Crispim


.. _blog_novo:

Blog novo, vida velha
=====================

Faz um tempo que eu já tava pensando: *"Pô, pra que banco de dados, servidor de
aplicação, um monte de coisa só pro meu blog? Seria muito melhor uns html queimado."*, mas
a preguiça sempre batia. Um dia fui pensar em colocar um feed vi que o blog ainda rodava
em python3.5. Ia dar um trabalhão pra atualizar! Então chegou a hora de um blog novo.


Um blog com sphinx
------------------

Já que eu queria uns html queimado, o mais simples seria usar o sphinx, que já estou
acostumado, é de boas escrever rst e já tinha um projeto de blog com sphinx, o
`ABlog <https://ablog.readthedocs.io/en/stable/>`_. Com isso já deu uma animada,
afinal eu não fui o único tonto que pensei em fazer um blog com sphinx.

A primeira coisa a fazer foi dar uma olhada no que o ABlog tem. E ele já tem
basicamente metade do que eu preciva: já criava listagem de posts automáticamente
e gerava feeds. Do que ele tem eu só precisaria mexer nas listas de posts pra paginar
a coisa. Depois de paginar as listas de posts era só eu fazer o mesmo esquema
que tem pros posts, só que pra fotos: Uma página pra cada foto e uma lista de fotos.
*Mamão-com-açúcar*.


É hora de fazer
---------------

Chegou a hora do *vamo vê* e aí eu vi. Primeiro a estrutura blog -> catalog -> collection.
Muito mais do que eu preciso, mas da hora. Depois tem o tal do init estranho que é
**_init** e não **__init__** que copia do __dict__ do cara. Serve pra um singleton
durante todo o build. Esperto! Sobrescrevi Blog (com PhotoBlog), Catalog, Collection
pra paginar as listas de posts, alterei a função que conecta no sinal do sphinx pra
gerar os html e uso uma opção a mais para mostrar o post todo na listagem ao invés de só
o começo. Foi de boas.

Agora era fazer uma listagem paginada de fotos. Alterei PhotoBlog pra ter uma collection
paginada de fotos, mas ainda me faltavam as fotos. Criei uma *directive* ``Photo`` e aí
faltava juntar uma coisa com a outra. Quando comecei a olhar como o ablog fazia isso
foi quando começou minha tristeza. A função do ablog que faz isso é gigante, chatona
de ler e o pior, é uma função muito ensimesmada, digamos assim, então não dá muito pra
extender a parada. E essa não é a única, tem mais algumas peças nesse estilo. A preguiça
começou a bater, mas vamos lá...

O esquema era basicamente pegar todos os `nodes` do tipo foto, guardar isso no env do
build e depois registrar nos PhotoBlog e boas, temos uma coleção de fotos no blog. Depois
disso extendi o **dirhtml** builder do sphinx pra copiar os arquivos das fotos pro diretório
de imagens do build e gerar as *thumbnails* que serão exibidas na lista de fotos.
Nesse ponto eu já tinha a maioria das funcionalidades que eu queria, era hora de mexer
no css. Aí a preguiça chegou de vez!


Os comentários
--------------

Eu fiquei um tempão sem mexer no blog, fazendo coisas mais úteis com meu tempo live
(tipo jogar xadrez ou tocar guitarra :P) até que eu lembrei que todo blog precisa de
comentários e animei de fazer os comentários.

.. note::

   O ABlog já tinha integração com o disqus e com o `Isso <https://isso-comments.de/>`_,
   que é basicamente o que eu precisava, mas o disqus eu não queria usar e a
   velha mania de não ler a documentação direito me fez não ver que tinha integração com o Isso.
   Na verdade nunca tinha ouvido falar dele, então acabei fazendo o meu mesmo. Tonto tontando.

A ideia era fazer uma coisa bem simples pros comentários, então decidi fazer o esquema
usando o sqlite de banco e uma api onde eu só preciso incluir um js na página e os cometários
já aparecem. Até aí nada demais, umas tabelinhas no sqlite, um endpoint pra criar comentário,
endpoint pra listar comentários (aqui dois na verdade, um que retorna um json e um que retorna
um  html), meia dúzia de linha de js (como o js deveria ser usado), o arroz com feijão da coisa.
Até que o `Bubbletea <https://github.com/charmbracelet/bubbletea>`_ me chamou a atenção.

O esquema dos comentários - chamado **parlante** - funciona com clientes que tem permissão
para certos domínios, então eu precisava cadastrar clientes e domínios e nada mais natural(?!)
que fazer uma **text user interface** pra isso, não? Foi assim que o que era pra ser só
uma paradinha de nada passou a ter dois executáveis: o **parlante** e o **parlane-tui**.
Já não bastava ter feito o que não precisava, fez o desnecessário duas vezes! Mas tudo bem,
foi. E agora eu tinha o que precisava pro blog. Ou quase...


Chegando nos finalmente
-----------------------

Depois de mais um tempão sem mexer no blog, peguei uns diazinhos de férias e decidi
terminar essa parada. Dei um tapa na tui pra ficar mais bonitinha e foi pro css do blog.
Eu já tinha o tema do sphinx que eu usava nas minhas documentações, a ideia era usar o
mesmo tema pro blog, era só dar um tapa e mesmo assim e encheu o saco. A página de foto
me deu uma canseira, foi difícil decidir como ela deveria ficar e depois demorei pra
descobrir como usar a coisa do **display:flex** com **order**. Depois de mexer no tema
me faltava o feed das fotos e um feed geral dos posts e fotos.

Comecei fazendo feed das fotos, que o conteúdo é basicamente o títudo da foto e a imagem.
Depois foi fazer o feed geral e nesse momento eu me encontrei de novo com uma das funções
feitas pra si mesma no ABlog, a parte que gera o feeds. No feed geral o item pode ser
uma foto ou um post. Consegui reusar o item do feed de fotos, já o de posts não deu, tive
que fazer um item de feed pra post específico pro feed geral. Passando por isso eu achava
que já tava tudo pronto. Dei aquele rsync com o servidor e comecei a navegar no blog pra
ver se tinha faltando algo. Tudo bem, tudo bom, até que a hostinger vem me atrapalhar.


A infra é sempre contra nóis, né?
---------------------------------

Enquanto eu tava dando uma (o que eu esperava ser) útima olhada na parada, do nada o blog
parou de responder, não pingava nem nada. No painel da hostinger a vps de pé, dava pra usar
pelo ssh web dos cara, as regra de iptables tudo normal. Que porra que tava acontecendo?
Troquei a minha conexão de casa pela do telefone e pimba! Foi! Fazendo um traceroute da coisa
vi que quem tava dropando meus pacote era o úlitmo hop antes de chegar na minha vps, era a
hostinger bloqueando meu ip! Agora passar pelo suporte que ia ser a dificuldade.

Primeiro tem que passar pelo bot maldito que te dá todos os passos que eu já tinha feito,
depois quando chega num humano a resposta padrão é sempre a mesma: vps é um serviço
auto gerenciado e a gente não pode fazer nada. Até alguém prestar atenção e entender
que o bloqueio era por parte da hostinger foi difícil. Depois que entenderam, o chamado
foi pra outra equipe, com outro prazo de atendimento. Depois de dois dias recebi um
email dizendo que tinham feito alterações no firewall da hostiger e eu não deveria ter
mais problemas. Funcionou até não funcionar mais.

Quando fui copiar o texto da página **Sobre** do blog antigo eu me lembrei que tinha
um formulário de contato lá que na verdade nunca funcionou porque não tinha nem botão
pra enviar a mensagem. Já que eu tava fazendo um blog novo essa era a hora de fazer
funcionar. Implementei o contato no parlante e na hora de subir a nova versão, cadê
que eu conseguia chegar no servidor? Bloquearam meu ip de novo! Vai lá eu falar com
o suporte, toda aquela coisa (esse vai e volta com infra já tava me lembrando da firma)
e até agora a coisa continua zuada.


Chegamos ao final
-----------------

Depois de mais de uma semana desisti da hostinger. Fiz uma conta no oracle cloud e
este bloguinho agora está no que diz ser uma vps pra sempre grátis. A máquina é bem
meia boca, mas como só tem os html do blog e o parlante, dá e sobra. Depois de toda
essa odisseia agora eu tenho um feed. Uau!

Ganhei algo com isso? Não. Vou ganhar algo com isso? Não. O blog é novo, mas a vidinha
continua a mesma.
