.. post:: Oct 31, 2024
   :category: computisses
   :author: Juca Crispim



Uma passada de olhos em websockets
==================================


Esses dias implementando websockets no
`tupi-proxy <https://github.com/jucacrispim/tupi-proxy>`_ e precisava de um
cliente e um servidor websocket pra poder testar e ao invés de pegar algo pronto eu
escrevi o que eu precisava. Então pra não ficar parecendo que foi um trabalho inútil,
vou escrever sobre websockets agora.

Websockets são uma conexão tcp normal onde assim que a conexão é estabelecida o cliente
envia os headers de uma requisição http, com os headers
``Upgrade: websocket`` e ``Connection: upgrade``.
Ao receber esses headers o servidor responde com esses mesmo headers indicando que suporta
websockets. Depois disso o cliente e servidor podem trocar mensagens.

Mas como assim?
---------------

Bom, primeiro o servidor tem que estar escutando em uma porta, aí o cliente faz uma
conexão tcp e manda uma requisição http GET para o servidor. Assim:

.. code-block:: http

   GET /ws HTTP/1.1
   Host: myhost.net:8000
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==


O header **Sec-WebSocket-Key** enviado pelo cliente é uma string de 16 caracteres
ascii aleatórios encodados em base64.

Ao receber essa requisição o servidor deve responder assim:

.. code-block:: http

   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=


O header **Sec-WebSocket-Accept** enviado pelo servidor é obtida da seguinte maneira: o
servidor concatena o **Sec-WebSocket-Key** enviado pelo cliente com a string mágica
"258EAFA5-E914-47DA-95CA-C5AB0DC85B11", gera um hash sha1 com essa string concatenada
e por fim encoda em base64.

Depois desse processo de handshake o servidor e o cliente devem manter a conexão
aberta e aí podem trocar mensagens usando o formato descrito na
`RFC 6455 <https://www.rfc-editor.org/rfc/rfc6455.html#section-5.2>`_.


O formato das mensagens
-----------------------

As mensagens trocadas via websockets são chamadas de frames. Os frames
enviados tanto pelo servidor quanto pelo cliente tem o seguinte formato:

.. code-block:: sh

   0                   1                   2                   3
	 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
	+-+-+-+-+-------+-+-------------+-------------------------------+
	|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
	|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
	|N|V|V|V|       |S|             |   (if payload len==126/127)   |
	| |1|2|3|       |K|             |                               |
	+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
	|     Extended payload length continued, if payload len == 127  |
	+ - - - - - - - - - - - - - - - +-------------------------------+
	|                               |Masking-key, if MASK set to 1  |
	+-------------------------------+-------------------------------+
	| Masking-key (continued)       |          Payload Data         |
	+-------------------------------- - - - - - - - - - - - - - - - +
	:                     Payload Data continued ...                :
	+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
	|                     Payload Data continued ...                |
	+---------------------------------------------------------------+


Essa coisa feia aí quer dizer que cada frame é formado da seguinte maneira:

* Primeiro byte:
    - bit 0: Indica se o frame é um frame final de mensagem ou se a mensagem
      será continuada em outro frame
    - bits 1 a 3: Bits revervados à extensões.
    - bits 4 a 7: Opcode

* Segundo byte:
    - bit 0: Indica se uma máscara está sendo usada
    - bits 1 a 7: O tamanho do payload

* Próximos dois bytes:
    - O tamanho do payload se o tamanho no segundo byte for >= 126.

* Próximos 8 bytes:
    - O tamanho do payload se o tamanho no segundo byte for == 127

* Próximos 4 bytes:
    - A máscara se uma estiver sendo usada

O restante dos bytes (até o tamanho do payload) é o payload.


Opcodes e máscara
-----------------

Os opcodes dão informação sobre o tipo do payload ou podem ser opcodes de
controle. O opcode 0 indica que a mensagem é uma continuação da mensagem no
frame anterior e o payload desse frame deve ser combinado com o payload do
anterior; o opcode 1 indica que o payload é um texto encodado em utf-8; o
opcode 2 indica que o payload é um binário;  o opcode 8 é um opcode de
controle usado para encerrar a conexão; o opcode 9 é um opcode de
controle para ping e por fim o opcode 10 é um opcode de controle usado para
pong.

A máscara são 32 bits aletórios que que vão encriptar os dados usando XOR.
Os clientes obrigatóriamente devem usar máscara quando enviando dados pro
servidor e o servidor não deve usar máscara quando enviando mensagens ao
cliente.

Bom, é basicamente isso o protocolo de websockets. Agora ao que importa.


Uma implementaçãozinha
----------------------

Primeiro uma implementação para wire encode e wire decode que vai ser usada
tanto pelo cliente quanto pelo servidor.

.. code-block:: go

   // Frame é como a gente envia mensagens através do websocket.
   // Essa struct representa aquele desenho feio lá de cima.
   type Frame struct {
       Opcode   byte
       Len      uint
       Payload  []byte
       Mask     []byte
       IsFinal  bool
       IsMasked bool
   }

   // WebSocket contém as operações básicas do protocolo
   // performadas tanto pelo cliente quanto pelo servidor.
   // Note que WebSocket não tem um método para criar uma
   // conexão já que a conexão sempre tem que ser criada
   // pelo cliente e nunca pelo servidor
   type WebSocket struct {
       Conn net.Conn
   }

   // Send wire encode um frame e envia os bytes em uma conxão
   // já aberta
   func (ws *WebSocket) Send(fr *Frame) error {
       data, err := ws.WireEncode(fr)
       if err != nil {
	   return err
       }

       _, err = ws.Conn.Write(data)
       return err
   }

   // Recv lê da conexão aberta e retorna o frame recebido.
   // Se Recv recebe um frame ping, envia um frame pong e
   // volta a ler da conexão. Se recebe um frame close
   // retorna um erro io.EOF.
   // Note que Recv não fecha a conexão.
   func (ws *WebSocket) Recv() (*Frame, error) {

       for {
	   fr, err := ws.WireDecode()

	   if err != nil {
	       return &Frame{}, err
	   }

	   switch fr.Opcode {
	   case OpcodeClose:
	       return &Frame{}, io.EOF

	   case OpcodePing:
	       fr.Opcode = OpcodePong
	       err := ws.Send(fr)
	       if err != nil {
		   return &Frame{}, err
	       }

	   default:
	       return fr, nil

	   }
       }
   }

   // RecvPayload retorna todo o payload da mensagem. Se a mensagem
   // estiver divida em mais de um frame, lê todos os frames e
   // só aí retorna o payload completo
   func (ws *WebSocket) RecvPayload() ([]byte, byte, error) {
       var unfinishedPayload []byte
       unfinishedOpcode := byte(0xFF)
       for {
	   fr, err := ws.Recv()
	   if err != nil {
	       return []byte{}, 0, err
	   }
	   if !fr.IsFinal {
	       unfinishedPayload = append(unfinishedPayload, fr.Payload...)
	       if unfinishedOpcode == 0xFF {
		   unfinishedOpcode = fr.Opcode
	       }
	       continue
	   }

	   if fr.Opcode == OpcodeCont {
	       unfinishedPayload = append(unfinishedPayload, fr.Payload...)
	       return unfinishedPayload, unfinishedOpcode, nil
	   }
	   return fr.Payload, fr.Opcode, nil

       }
   }

   // Close manda um frame de controle close e fecha a conexão.
   func (ws *WebSocket) Close() error {
       msg := []byte("close connection")
       fr := Frame{
	   Opcode:  OpcodeClose,
	   Payload: msg,
	   Len:     uint(len(msg)),
	   IsFinal: true,
       }
       ws.Send(&fr)
       return ws.Conn.Close()
   }

   // WireEncode transforma um frame em uma sequencia de bytes
   // que vai ser enviada pela conexão.
   // WireEncode não força o uso de máscara
   func (ws *WebSocket) WireEncode(fr *Frame) ([]byte, error) {
       data := make([]byte, 2)

       if fr.IsFinal {
	   // aqui se o frame for o frame final de uma mensagem
	   // a gente seta o primeiro bit pra zero.
	   data[0] = 0x00
       } else {
	   // se não for um frame final a gente seta pra 1
	   data[0] = 0x80
       }
       // os quatro últimos bits do primeiro byte são
       // o opcode
       data[0] |= fr.Opcode

       l := len(fr.Payload)

       if l <= 125 {
	   // se o payload for menor que 126 bytes
	   // o temanho será os últimos 7 bits do
	   // primeiro byte
	   data[1] = byte(l)

       } else if float64(l) < math.Pow(2, 16) {
	   // se o tamanho do payload couber em dois bytes a gente
	   // marca os sete últimos bits do segundo byte como 126
	   // e marca o tamanho do payload nos próximos dois.
	   data[1] = byte(126)
	   s := make([]byte, 2)
	   binary.BigEndian.PutUint16(s, uint16(l))
	   data = append(data, s...)
       } else if float64(l) < math.Pow(2, 64) {
	   // se o tamanho do payload cabe em oito bytes marcamos
	   // nos próximos 8
	   data[1] = byte(127)
	   s := make([]byte, 8)
	   binary.BigEndian.PutUint64(s, uint64(l))
	   data = append(data, s...)
       } else {
	   // muito grande. tem que dividir a mensagem em
	   // mais de um frame
	   return []byte{}, errors.New("Payload muito grande")
       }

       if fr.Mask != nil && len(fr.Mask) > 0 && len(fr.Mask) != 4 {
	   return []byte{}, errors.New("Invalid mask")
       }
       if fr.Mask != nil && len(fr.Mask) == 4 {
	   // Se uma mascara é usada setamos o primeiro bit
	   // do segundo byte para 1 e fazemos o XOR no payload
	   data[1] = 0x80 | data[1]
	   data = append(data, fr.Mask...)
	   xOR(fr.Payload, fr.Mask)
       }
       // e por fim o payload depois da tralha toda
       data = append(data, fr.Payload...)
       return data, nil
   }

   // WireDecode lê da conexão aberta e retorna o frame recebido.
   // Aqui a gente tá basicamente fazendo o contrário do que fizemos
   // em WireEncode
   func (ws *WebSocket) WireDecode() (*Frame, error) {
       fr := Frame{}
       d := make([]byte, 2)
       _, err := ws.Conn.Read(d)
       if err != nil {
	   return nil, err
       }

       // verificando se o primeiro bit é 0 ou 1 pra saber
       // se é um frame final. 0 == final
       final := (d[0] & 0x80) == 0x00

       // Pegando os últimos 4 bits do primeiro byte que
       // são o opcode
       opcode := d[0] & 0x0F

       // Primeiro byte indica se tá usando máscara ou não
       // 1 == tá usando
       isMasked := (d[1] & 0x80) == 0x80

       // os 7 últimos bits do segundo byte pro tamanho do
       // payload. Se for <= 125 já será o tamanho real
       len := d[1] & 0x7F
       l := uint(len)

       fr.Opcode = opcode
       fr.IsFinal = final
       fr.IsMasked = isMasked

       if l == 126 {
	   // se o marcado no segundo byte é 126 então o tamanho
	   // está nos próximos dois bytes
	   d := make([]byte, 2)
	   _, err := ws.Conn.Read(d)
	   if err != nil {
	       return nil, err
	   }
	   l = uint(binary.BigEndian.Uint16(d))
       } else if l == 127 {
	   // se o marcado no segundo byte é 127 então o tamanho
	   // está nos próximos 8 bytes
	   d := make([]byte, 8)
	   _, err := ws.Conn.Read(d)
	   if err != nil {
	       return nil, err
	   }
	   l = uint(binary.BigEndian.Uint64(d))
       }

       fr.Len = l

       mask := make([]byte, 4)
       if isMasked {
	   // se tá usando máscara, os próximos 4 bytes serão
	   // a máscara.
	   _, err = ws.Conn.Read(mask)
	   if err != nil {
	       return nil, err
	   }
       }

       // e por fim o payload do frame
       payload := make([]byte, l)
       _, err = ws.Conn.Read(payload)

       if isMasked {
	   xOR(payload, mask)
	   fr.Mask = mask

       }
       fr.Payload = payload
       return &fr, nil
   }


Agora o código do websocket client:

.. code-block:: go



   // WebSocketClient é quem inicia a conexão de websocket.
   type WebSocketClient struct {
       WebSocket
       URL *url.URL
   }

   // Handshake envia uma requisição http com headers upgrade
   // perguntando se o servidor suporta websockets
   func (ws *WebSocketClient) Handshake() error {
       // O hash aqui são 16 caracteres ascii aleatórios encodados
       // em base64
       hash := getSecHashClient()
       req := &http.Request{
	   URL:    ws.URL,
	   Header: make(http.Header),
       }

       req.Header.Set("Upgrade", "websocket")
       req.Header.Set("Connection", "upgrade")
       req.Header.Set("Sec-WebSocket-Accept", hash)

       err := req.Write(ws.WebSocket.Conn)
       if err != nil {
	   return err
       }
       reader := bufio.NewReaderSize(ws.Conn, 4096)
       resp, err := http.ReadResponse(reader, req)
       if err != nil {
	   return err
       }

       // O status que o servidor deve retornar informando
       // que suporta websockets é o status 101
       if resp.StatusCode != http.StatusSwitchingProtocols {
	   return errors.New("Server does not support websockets")
       }

       if strings.ToLower(resp.Header.Get("Upgrade")) != "websocket" ||
	   strings.ToLower(resp.Header.Get("Connection")) != "upgrade" {
	   return errors.New("Invalid response")
       }
       return nil
   }

   // Send envia um frame ao servidor e antes de enviar
   // gera uma máscara para o frame
   func (ws *WebSocketClient) Send(fr *Frame) error {
       fr.Mask = getMask()
       return ws.WebSocket.Send(fr)
   }

   // NewWebSocketClient retorna um cliente de websocket já
   // connectado a um servidor que suporta websockets
   func NewWebSocketClient(rawURL string) (*WebSocketClient, error) {
       u, err := url.Parse(rawURL)
       if err != nil {
	   return &WebSocketClient{}, nil
       }

       hostPort, err := getHostPort(u)
       if err != nil {
	   return &WebSocketClient{}, err
       }

       conn, err := net.Dial("tcp", hostPort)
       if err != nil {
	   return &WebSocketClient{}, err
       }
       ws := WebSocketClient{
	   WebSocket: WebSocket{
	       Conn: conn,
	   },
	   URL: u,
       }

       err = ws.Handshake()
       if err != nil {
	   return &WebSocketClient{}, err
       }

       return &ws, nil
   }


Agora o server que a única coisa que faz é retornar o que o cliente mandar, mas
se for um texto retorna o texto invertido:

.. code-block:: go

   // WebSocketServer responde a uma conexão feita pelo cliente.
   type WebSocketServer struct {
       WebSocket
       Header http.Header
   }

   // Handshake retorna status 101 indicando que aceita websockets
   func (ws *WebSocketServer) Handshake() error {
       secKey := ws.Header.Get("Sec-WebSocket-Key")
       hash := getSecHashServer(secKey)
       headers := []string{
	   "HTTP/1.1 101 Switching Protocols",
	   "Upgrade: websocket",
	   "Connection: upgrade",
	   "Sec-WebSocket-Accept: " + hash,
	   "",
	   "",
       }
       _, err := ws.Conn.Write([]byte(strings.Join(headers, "\r\n")))
       return err
   }

   // Recv retorna o frame enviado pelo cliente. Se o cliente enviar
   // um frame sem máscara Recv retorna um erro já que o cliente
   // sempre tem que usar uma máscara
   func (ws *WebSocketServer) Recv() (*Frame, error) {
       fr, err := ws.WebSocket.Recv()

       if err != nil {
	   return fr, err
       }
       if !fr.IsMasked {
	   return &Frame{}, errors.New("Clients must mask the payload")
       }
       return fr, err
   }

   // Echo simplesmente retorna a mensagem enviada pelo cliente
   // invertendo a string se o payload for utf-8.
   func (ws *WebSocketServer) Echo() error {
       for {
	   payload, opcode, err := ws.RecvPayload()
	   if err != nil && errors.Is(err, io.EOF) {
	       log.Println("Connection closed")
	       return nil
	   }

	   if err != nil {
	       log.Println(err.Error())
	       return err
	   }

	   if opcode == OpcodeText {
	       runes := []rune(string(payload))
	       pl := len(runes)
	       reversed := make([]rune, pl)
	       for i := pl - 1; i >= 0; i-- {
		   j := (pl - 1) - i
		   reversed[j] = runes[i]
	       }
	       payload = []byte(string(reversed))
	   }

	   fr := Frame{
	       Payload: payload,
	       Opcode:  opcode,
	   }
	   err = ws.Send(&fr)
	   if err != nil {
	       log.Println(err.Error())
	       return err
	   }
       }
   }


E por fim pra testar as coisas tudo junto a gente faz um http handler e uma cli.

.. code-block:: go

   func wsCli() {

       ws, err := NewWebSocketClient("ws://localhost:8081")
       if err != nil {
	   panic(err.Error())
       }

       var msg string
       for {
	   fmt.Print(": ")
	   reader := bufio.NewReader(os.Stdin)
	   msg, err = reader.ReadString('\n')
	   if err != nil {
	       panic(err.Error())
	   }

	   frame := Frame{
	       Payload: []byte(msg),
	       IsFinal: true,
	       Opcode:  OpcodeText,
	   }
	   ws.Send(&frame)
	   resp, err := ws.Recv()
	   if err != nil {
	       panic(err.Error())
	   }
	   fmt.Printf(string(resp.Payload) + "\n")
       }

   }

   func wsHandler(w http.ResponseWriter, r *http.Request) {
       h, ok := w.(http.Hijacker)
       if !ok {
	   w.WriteHeader(http.StatusInternalServerError)
	   return
       }

       conn, _, err := h.Hijack()
       if err != nil {
	   log.Println(err.Error())
	   w.WriteHeader(http.StatusInternalServerError)
	   return
       }
       ws := WebSocketServer{
	   WebSocket: WebSocket{
	       Conn: conn,
	   },
	   Header: r.Header,
       }

       defer ws.Close()

       err = ws.Handshake()
       if err != nil {
	   log.Println(err.Error())
	   w.WriteHeader(http.StatusInternalServerError)
	   return
       }

       err = ws.Echo()
       if err != nil {
	   log.Println(err.Error())
	   w.WriteHeader(http.StatusInternalServerError)
	   return
       }
   }

A main function fica assim:

.. code-block:: go

   func main() {
       server := flag.Bool("server", false, "start the server")
       client := flag.Bool("client", false, "start the client")

       flag.Parse()

       if !*server && !*client {
	   panic("one of server or client must be true")
       }

       if *server && *client {
	   panic("only one of server and client can be true")
       }
       if *server {
	   log.Fatal(http.ListenAndServe(":8081", http.HandlerFunc(wsHandler)))
       } else {
	   wsCli()
       }
   }

Agora só compilar assim:

.. code-block:: sh

   $ go build -o ws ws.go


Inicie o servidor assim:

.. code-block:: sh

   $ ./ws -server

E agora você pode usar o cliente para falar com o servidor via websockets

.. code-block:: sh

   $ ./ws -client
   : olá, mundo

   odnum ,álo
   :


O código completo pode ser baixado `aqui <https://docs.poraodojuca.dev/ws.go>`_.

E é isso!
