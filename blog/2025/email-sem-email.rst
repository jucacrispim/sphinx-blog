.. post:: Aug 28, 2025
   :tags: computisses
   :author: Juca Crispim


.. |br| raw:: html

    <br/>

Enviar email sem enviar email
=============================

No :ref:`post anterior <blog_novo>` eu comentei que fiz um formulário de contato pro blog e
pra isso precisava me avisar quando chegasse mensagem. O mais óbvio seria mandar
um email pra mim mesmo, mas por que enviar um email quando eu posso só
salvar um arquivo num diretório?

Eu já conhecia por cima o maildir, que é um formato usado para armazenar
emails num sistema de arquivos local, onde cada email é um arquivo, tem
um *layout* específico de diretórios e regras para os nomes dos arquivos. A
`especificação do maildir <https://cr.yp.to/proto/maildir.html>`_ é bem pequena
e trata dos diretórios onde as mensagens devem ser salvas e como os nomes
dos arquivos devem ser gerados. Já o formato usado no arquivo com com o email
é definido pelas RFCs `5322 <https://datatracker.ietf.org/doc/html/rfc5322>`_,
`2045 <https://datatracker.ietf.org/doc/html/rfc2045>`_,
`2046 <https://datatracker.ietf.org/doc/html/rfc2046>`_ e
`2047 <https://datatracker.ietf.org/doc/html/rfc2047>`_.


O maildir
---------

A estrutura de diretórios do maildir são três diretórios: ``tmp``, ``cur`` e ``new``.
O diretório ``tmp`` é onde ficam os arquivos dos emails que ainda estão sendo
recebidos. Quando o arquivo é recebido ele tem que ser copiado para o diretório ``new``.
O cliente de emails avisa o usuário que existem novos emails. E quando o usuário
ler o email ele deve ser movido para o diretório ``cur``. O nome do arquivo deve ser
um nome único e quando movido de ``new`` para ``cur`` o nome deve ter adicionado
algumas informações no final.

.. note::

   Na especificação do maildir tem algumas sugestões para gerar o nome único,
   entre eles pegar algo de /dev/urandom e deixar o nome como o hexa do que
   foi pego em /dev/urandom. Basicamente pode-se usar um UUID4 (acho :P)

As informações adicionadas ao final do nome do arquivo depois de movido são basicamente
informações sobre o status do email. A semântica da info é a seguinte:

Info começada com ``1,``: Semântica experimental |br|
Info começada com ``2,``: Cada letra depois da vírgula é uma *flag* independente.

* As flags são:
    - ``P`` (passed): O usuário encaminhou o email para alguém
    - ``R`` (replied): O usuário respondeu o email
    - ``S`` (seen): O usuário viu o conteúdo do email
    - ``T`` (trashed): Mensagem marcada como lixo
    - ``D`` (draft): Mensagem é um rascunho
    - ``F`` (flagged): Uma tag definida pelo usuário

Então uma mensagem em um arquivo nomeado ``22eb0cf0-b71c-4411-88a1-aef292007e58`` quando
está em ``new``, depois de movido para ``cur`` teria um nome mais ou  menos assim
``22eb0cf0-b71c-4411-88a1-aef292007e58:2,SR``.


O IMF
-----

IMF é o *internet message format* que é um formato para transmissão de mensagens de texto
em ASCII definido na `RFC 5322 <https://datatracker.ietf.org/doc/html/rfc5322>`_ e
estendido com a introdução de MIME tyes pela
`RFC 2045 <https://datatracker.ietf.org/doc/html/rfc2045>`_ (e relacionados)
para utilização de outros conjuntos de caracteres e envio de outros tipos de
conteúdo além de texto, como imagens e sons.

Mensagens são basicamente duas partes: o cabeçalho e o corpo. O cabeçalho pode ter várias
linhas, que são os campos do cabeçalho, no formato: ``NomeDoCampo: Valor\r\n``. Uma
linha em branco separa o cabeçalho do corpo. O corpo na especificação original era
somente uma mensagem em ASCII e com a introdução de MIME types o corpo pode ser
uma variedade de formatos, definidos num cabeçalho. Aqui vai um exemplo de uma
mensagem com um corpo **multipart** que pode conter vários tipos de mensagem no
mesmo corpo

.. code-block:: text

   Date: Thu, 28 Aug 2025 01:54:01 -0300
   From: Juca <juca@poraodojuca>
   To: Zé <ze@casadoze>
   Subject: Olá
   MIME-Version: 1.0
   Message-ID: 1756357403.issodeveriaserumaidentificacao@poraodojuca
   Content-Type: multipart/mixed; boundary="uma-string-que-marca-o-limite"

   --uma-string-que-marca-o-limite
   Content-Type: text/plain; charset="UTF-8"

   Como vai, como vai, vai, vai?

   --uma-string-que-marca-o-limite
   Content-Type: text/html; charset="UTF-8"

   <html><body><strong>
   Como vai, como vai, vai, vai?
   </strong></body></html>

   --uma-string-que-marca-o-limite
   Content-Type: image/jpeg
   Content-Disposition: attachment; filename="foto.jpeg"
   Content-Transfer-Encoding: base64

   ... aqui viria uma imagem jpeg em base64 ...

   --uma-string-que-marca-o-limite--


O que acontece aí é o seguinte: A primeira parte, até a primeira linha em branco
é o cabeçalho da mensagem e o restante depois da primeira linha em branco é o corpo,
um corpo que tem 3 partes distintas, uma em texto puro, uma em html e uma imagem
anexa.

Os campos do cabeçalho obrigatórios são o ``From`` e o ``Date``, o ``Message-ID`` apesar
de não ser obrigatório deveria estar presente em todas as mensagens. O formato de
``Message-ID`` é ``<timestamp>.<uma-string-identificadora>@<umhost>``. O
``Content-Type: multipart/mixed; boundary="uma-string-que-marca-o-limite"``
indica que o corpo da mensagem está dividido em várias partes, cada um com seu próprio
Content-Type. ``boundary`` é uma string que separa uma parte do corpo de outra. Essa
string deve ser gerada de maneira que seja bem difícil ela esteja repetida no corpo
do email.


Eu não implementei tudo isso
----------------------------

Claro que isso é coisa demais só pro que eu precisava. O formulário de contato é só um campo de texto
e pra fazer a parte do maildir já tinha o `go-maildir <github.com/emersion/go-maildir>`_. Então a
minha implementaçãozinha meia-boca ficou mais ou menos assim:

.. code-block:: go

    // EmailMessage represents an email to be sent. Note that as this have
    // no content type and the body is a string, only text/plain bodies are
    // supported.
    type EmailMessage struct {
	    From      string
	    To        []string
	    Subject   string
	    Body      string
	    Timestamp int64
    }

    // NewEmailMessage checks for missing from or to.
    func NewEmailMessage(from string, to []string, subject string, body string) (EmailMessage, error) {
	    if from == "" {
		    return EmailMessage{}, errors.New("from can't be empty")
	    }
	    if to == nil || len(to) == 0 {
		    return EmailMessage{}, errors.New("to can't be empty")
	    }
	    ts := time.Now().Unix()
	    msg := EmailMessage{
		    From:      from,
		    To:        to,
		    Subject:   subject,
		    Body:      body,
		    Timestamp: ts,
	    }
	    return msg, nil
    }


    type keyGen func() (string, error)

    // MaildirSender represents a maildir delivery
    type MaildirSender struct {
	    MaildirPath string
	    keyGen      keyGen
    }

    // SendEmail writes an EmailMessage to a local maildir
    func (s MaildirSender) SendEmail(msg EmailMessage) error {
	    var d = maildir.Dir(s.MaildirPath)
	    err := initMaildir(d)
	    if err != nil {
		    return err
	    }

	    mformat, err := EmailMessage2Maildir(msg, s.keyGen)
	    if err != nil {
		    return err
	    }

	    del, err := maildir.NewDelivery(s.MaildirPath)
	    if err != nil {
		    return err
	    }

	    _, err = del.Write([]byte(mformat))
	    if err != nil {
		    return err
	    }

	    err = del.Close()
	    if err != nil {
		    return err
	    }
	    return nil
    }

    // NewMaildirSender returns a new NewMaildirSender instance
    func NewMaildirSender(path string) MaildirSender {
	    s := MaildirSender{
		    MaildirPath: path,
		    keyGen:      GenKey,
	    }
	    return s
    }

    // EmailMessage2Maildir converts an EmailMessage to a string in the
    // maildir file format.
    func EmailMessage2Maildir(msg EmailMessage, gen keyGen) (string, error) {

	    dtfmt := "Mon, 2 Jan 2006 15:04:05 -0700"
	    loc, err := time.LoadLocation("UTC")
	    if err != nil {
		    return "", err
	    }
	    dt := time.Unix(msg.Timestamp, 0).In(loc)
	    dtStr := dt.Format(dtfmt)
	    key, err := gen()
	    if err != nil {
		    return "", err
	    }
	    msgId := fmt.Sprintf("<%d.%s@localhost>", msg.Timestamp, key)

	    mformat := fmt.Sprintf("From: %s\n", msg.From)
	    toStr := strings.Join(msg.To, ",")
	    mformat += fmt.Sprintf("To: %s\n", toStr)
	    mformat += fmt.Sprintf("Subject: %s\n", msg.Subject)
	    mformat += fmt.Sprintf("Date: %s\n", dtStr)
	    mformat += fmt.Sprintf("Message-ID: %s\n", msgId)
	    mformat += fmt.Sprintf("MIME-Version: 1.0\n")
	    mformat += fmt.Sprintf("Content-Type: text/plain; charset=\"UTF-8\"\n")
	    mformat += "\n"
	    mformat += msg.Body
	    return mformat, nil
    }

Mais pra frente do post vai ficar claro que eu não precisava do go-maildir, mas na hora foi isso
o que eu fiz e boas, ficou! Agora é só meter um rsync pra pegar esses arquivos pra minha máquina
e foi.


Claro que tinha um bug no Kmail
-------------------------------

O Kmail é um cliente de mail pro KDE que tem muitos anos que eu sempre tento dar uma chance.
Fui dar mais uma chance com o maildir, e claro que tinha um bug. Então chegou a hora da
jigajoga!

A coisa é assim: na minha máquina local eu tenho uma estrutura de diretórios do maildir, com
``new``, ``cur`` e ``tmp``, assim o Kmail já reconhece esse diretório como um diretório que
vai receber emails. Aí eu faço um rsync do diretório ``new`` do servidor para o diretório
``new`` na minha máquina e movo as menagens para um diretório de backup no servidor.

.. note::

   Aqui que a coisa fica clara que eu não precisava do maildir no servidor, era só
   jogar o arquivo num diretório qualquer, mas enfim... Burrice nunca falta no estoque.


O bug é que depois que eu fazia o rsync dos emails não atualizava o Kmail automaticamente,
eu precisava clicar em "Atualizar" pra receber uma notificação. Então pra finalizar, depois
do rsync, a gente dá um chute no Kmail pra ele acordar.

.. code-block:: shell

    #!/bin/bash

    LOCALDIR=~/somewhere/new/
    REMOTEDIR='/somewhere/new/'
    REMOTEDIR_CUR='/somewhere/cur/'

    rsync -avz --progress eu@meuservidor:$REMOTEDIR $LOCALDIR --rsync-path="sudo rsync"
    ssh eu@meuservidor "sudo find $REMOTEDIR -maxdepth 1 -type f -exec mv {} $REMOTEDIR_CUR \;"

    # aqui a gente usa o dbus e pega todos os recursos que estão vinculados ao Akonadi
    qdbus org.freedesktop.Akonadi /ResourceManager org.freedesktop.Akonadi.ResourceManager.resourceInstances |
	while read resource;
	do

	    if [[ "$resource" == *"maildir"* ]]; then
	        # se for maildir, a gente manda sincronizar a coisa
		SERVICE_DBUS="org.freedesktop.Akonadi.Resource.${resource}"
		OBJECT_PATH="/"
		METHOD_NAME="org.freedesktop.Akonadi.Resource.synchronize"
		qdbus $SERVICE_DBUS $OBJECT_PATH $METHOD_NAME

	    fi
	done

E agora sim, envio e recebo emails sem enviar emails!
