Emacs é o editor mais legal
===========================

.. post:: Oct 19, 2024
   :category: computisses
   :author: Juca Crispim


Eu uso emacs desde o terceiro dia do meu primeiro emprego. Sempre pensei em
escrever algo sobre aqui, mas aí sempe ficava naquelas: "Põ, é só um editor de
texto, besteira escrever sobre isso". Mas esses dias no trampo fiz um
esqueminha que na minha opnião é a melhor coisa do emacs: Elisp e poder fazer
qualquer coisa com seu editor


É só executar uma query, caralho
---------------------------------

Se sempre que vão contar a história do software livre metem uma história com
impressora no meio, não tem problema nenhum minha história ser sobre só
executar umas query, né?

A coisa começa com um serviço que temos lá no trampo e eu tinha feito umas apis
pra consulta, o normal, tipo:

.. code-block:: http

   GET /api?fields=a,sum(1)&filter=b__gt=2|c=3&group=a HTTP/1.1


O problema com isso que é quando isso começa a crescer a coisa fica feia pra
cacete e muito disso era feita na mão, alguém fazia uma query, mandava via curl
ou postman e pegava o json da resposta. Então era muito ruim pro pessoal
escrever isso. Um dia um colega me mostrou a api do salesforce que você
basicamente manda um query sql na api. Aí implementei um sql parser (não sou
doido de deixar umas query aberta na minha api) e começaram a uma api com um
sql direto.

Isso acabou sendo bem mais prático de usar, de explicar pros outros, então a
mesma coisa começou a ir pra outros serviços e em pouco tempo eu tinha uma meia
dúzia de serviços com apis sql e aí acabou ficando um saco fazer as queries
todas via postman ou curl.

Minha primeira solução foi fazer um cliente em python + readline que ficou
melhor de usar do que a maneira que eu usava antes, mas ainda faltava
coisa. Eu basicamente queria highlight syntax pro sql e pro json do retorno.
O CodePrettifier faz highlight systax, mas só gera html e eu teria que mudar um
monte de coisa. Trabalho demais pra uma coisa bem pouco útil.


Lá vem o Emacs
--------------

Nesse ponto eu me liguei: o emacs já tem o que eu quero, usemo-lo! E com isso
eu precisava fazer bem menos coisa, só alterar o cliente em python que tinha
feito para só aceitar uma query como parâmetro, fazer o request e cuspir o
retorno (eu poderia fazer o request pra api de dentro do emacs, mas o outro já
estava pronto, com o esquema de autenticação e talz, entõa assim seria mais
fácil). Do lado do emacs seria só pegar uma query do buffer que eu estou
usando, chamar o cliente com a query como param e pegar o retorno.

Explicando melhor o que foi a minha ideia aqui: eu escrevo as queries em um
buffer em sql-mode no emacs pra poder ter o realce de sintaxe e quando eu
usasse um atalho essa query seria executada, a janela dividia em duas e o
retorno seria exibido ao lado do do buffer onde eu digito a query.

O código pra fazer isso ficou assim:

.. code-block:: elisp

   (defun sequela-exec ()

     (interactive)

     ;; aqui a gente pega o texto do buffer atual
     (setq sql (buffer-substring-no-properties (point-min) (point-max)))

     ;; aqui a gente pega uma linha do tipo
     ;; env: SomeWhere
     ;; onde SomeWhere é o serviço que a gente quer executar aquery
     (string-match "^env: (.*)" sql)
     (setq sequela-env (match-string 1 sql))
     (unless sequela-env
       (user-error "missing env"))

     ;; deixando só a query que vamos excutar, tirando comentários, env
     ;; e new lines.
     (setq query (replace-regexp-in-string "--.*$\\|^env:.*$\\|\n" "" sql))

     (setq buffname "sequela-idle")
     (setq proc-buffname "sequela")

     ;; aqui divide a janela em duas e vai para a janela onde será exibido
     ;; o resultado
     (pop-to-buffer buffname)
     (erase-buffer)
     (insert "sequelando...")

     (with-current-buffer (get-buffer-create proc-buffname)
       (erase-buffer))

     ;; executa o script que faz o acesso à api
     (start-process proc-buffname proc-buffname "query.py" sequela-env query)

     ;; o sentinela do processo é chamado quando o estado do processo muda
     (set-process-sentinel
      (get-process proc-buffname)
      (lambda (process event)
	(when (string= event "finished\n")
	  ;; aqui quando o processo terminar a gente formata o
	  ;; resultado e copia para o buffer onde é exibido o resultado
	  (with-current-buffer proc-buffname
	    (json-pretty-print-buffer)
	    (copy-to-buffer (get-buffer-create buffname) (point-min) (point-max)))
	  (pop-to-buffer buffname)
	  (json-ts-mode)))))

Legal, né? Agora é só eu abrir um arquivo .sql, escrever a query que eu quiser,
executar essa função e pronto! Um hackzinho vagabundo no emacs é mais fácil e
fica melhor do que fazendo um programa separado!
