.. post:: Nov 12, 2024
   :tags: computisses
   :author: Juca Crispim


Uma jigajoga bacana
===================

**Jigajoga** \|ó\| (ji-ga-jo-ga) - Artifício, ludíbrio; mecanismo ou solução
resultante de improvisação. Depois que eu aprendi essa palavra eu nunca mais
consegui dizer hack ou gambiarra, só jigajoga. E hoje vou contar de uma
jigajoga do trampo.


O galho
-------

Esses tempos no trampo eu precisava identificar um usuário que chega no nosso
whatsapp. Em geral pra mandar alguém pro whatsapp você só manda url
``https://wa.me/<telefone>?text=oi``. O problema aí é que eu não faço ideia de
como o usuário chegou no whatsapp. Ele pode ter acessado via um link ou ter
simplesmente chegado no whatsapp e falado oi. A única coisa que consigo saber
com isso é o texto que o usuário me mandou e o número do telefone dele.

A primeira parte da coisa é simples, ao invés de enviar o usuário para o link
do whatsapp direto, manda um link pra mim, aí eu consigo gerar um fingerprint
do usuário e depois redirecionar para o whatsapp. Mas depois que mandar o
usuário pro whatsapp, como eu sei que o usuário que chegou lá é o usuário x?


A ideia
-------

Logo de cara um colega me mostrou como outra empresa tava fazendo isso:
mandando uma string XYZ na mensagem, pedindo para o usuário enviar essa
mensagem com a string e essa string serviria de id pra identificar o usuário.
Mas isso é feio pra caralho. Então minha primeira ideia foi usar caracteres
'invisíveis' (non-printable) no meio da mensagem. E bom, se eu precisava de uma
identificação única de usuário o óbvio seria uuid, no caso o 4.

Então peguei uma lista de 17 caracteres 'invisíveis' (16 pra a-f e 1 pra '-')
e aí fazia a tradução da representação em string de um uuid v4 pra uma string
invisível usando os caracteres non-printable. Subi a coisa rapidinho, testei no
meu whatsapp, funcionou. Coisa linda.

O código ficou mais ou menos assim:

.. code-block:: python

   # -*- coding: utf-8 -*-

   NON_PRINTABLE_CHARS = [
       '\u200b',
       '\u2060',
       '\u2061',
       '\u2062',
       '\u2063',
       '\u2064',
       '\u2066',
       '\u2067',
       '\u2068',
       '\u2069',
       '\u206A',
       '\u206B',
       '\u206C',
       '\u206D',
       '\u206E',
       '\u206F',
       '\uFE06',
   ]

   PRINTABLE_CHARS = '0123456789abcdef-'


   def translate_uuid_to_invisible(u):
       inv = ''
       ustr = str(u)
       for i in range(len(ustr)):
	   inv += NON_PRINTABLE_CHARS[PRINTABLE_CHARS.index(ustr[i])]

       return inv

   def translate_uuid_from_invisible(inv):
       u = ''
       for i in range(len(inv)):
	   u += PRINTABLE_CHARS[NON_PRINTABLE_CHARS.index(inv[i])]

       return u

   def put_fingerprint(text, uuid):
       inv = translate_uuid_to_invisible(uuid)
       t = text[0] + inv + text[1:]
       return t

   def get_fingerprint(t):
       if t[1] not in NON_PRINTABLE_CHARS:
	   return ''

       inv = t[1:37]
       u = translate_uuid_from_invisible(inv)
       return u


   if __name__ == '__main__':
       from uuid import uuid4

       txt = 'Olá, mundo!'
       u = uuid4()

       t = put_fingerprint(txt, u)
       fp = get_fingerprint(t)

       assert str(u) == fp


Claro que nunca funciona de primeira
------------------------------------

Depois com mais testes percebi que eu tinha um problema: no whatsapp web, a
depender do uuid, ficava uns espaços em branco no meio do texto, algo tipo
`o   i`. e isso por causa da combinação de caracteres. Apesar dos caracteres
que eu escolhi não terem uma representação, eles tem uma função e a maioria
deles eu nem sei qual a função.

Pra corrigir isso, eu primeiro tentei ir substituindo os caracteres por outros
até que desse certo, mas é um trabalhão danado, sem change. Então eu precisava
diminuir o número de caracteres usados pra ficar mais fácil a coisa de tirar os
espaços do whatsapp web.


Menos (caracteres) é mais (espaço)
----------------------------------

Um uuid é um número de 128 bits, então eu pensei em escrever isso em
'binário', aí eu só precisaria de dois caracteres, mas em contrapartida eu
teria 128 caracteres a mais em cada mensagem. Nada é grátis, mas era mais
importante o texto ficar 'certo' do que o tamanho da mensage.

Escolhi os caracteres \u200b (zero width space) e \u2060 (zero width word
joiner), alerei o código pra pegar a string com a representação do número do
uuid em binário e simplesmente troquei 0 e 1 pelos caracteres zero witdh
escolhidos. Altera o código, sobre rapidinho, testa, testa, testa... Bada bim!
Bada bam! Bada bum! Dessa fez funcionou. Merge no master, pusha e vai! Daqui a
uns minutinhos tá em prd!

O código alterado ficou mais ou menos assim:

.. code-block:: python

   # -*- coding: utf-8 -*-

   from uuid import UUID

   NON_PRINTABLE_CHARS = [
       '\u200b',
       '\u2060',
   ]

   PRINTABLE_CHARS = '01'


   def translate_uuid_to_invisible(u):
       inv = ''
       bitstr = f'{u.int:128b}'.replace(' ', '0')
       for i in range(len(bitstr)):
	   inv += NON_PRINTABLE_CHARS[PRINTABLE_CHARS.index(bitstr[i])]

       return inv

   def translate_uuid_from_invisible(inv):
       bitstr = ''
       for i in range(len(inv)):
	   bitstr += PRINTABLE_CHARS[NON_PRINTABLE_CHARS.index(inv[i])]

       n = int(bitstr, 2)
       u = UUID(int=n)
       return u

   def put_fingerprint(text, uuid):
       inv = translate_uuid_to_invisible(uuid)
       t = text[0] + inv + text[1:]
       return t

   def get_fingerprint(t):
       if t[1] not in NON_PRINTABLE_CHARS:
	   return ''

       inv = t[1:129]
       u = translate_uuid_from_invisible(inv)
       return u


   if __name__ == '__main__':
       from uuid import uuid4

       txt = 'Olá, mundo!'
       u = uuid4()

       t = put_fingerprint(txt, u)
       fp = get_fingerprint(t)

       assert u == fp


Ainda tem problemas, 128 caracteres a mais pra um simples "oi" é feio, se
eu printar isso num terminal ainda fica um espaço entre as letras, mas não
é pra usar no terminal mesmo... E isso é uma jigajoga, não dá pra esperar
muito, só que resolva o problema em mãos.

Moral da história: Não tem!
