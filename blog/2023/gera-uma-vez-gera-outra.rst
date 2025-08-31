Gera uma vez, gera outra!
=========================

.. post:: May 11, 2023
   :category: computisses
   :author: Juca Crispim



Bom, seguindo com a minha ideia de escrever mais no blog nem que seja inútil,
hoje vou escrever sobre geradores em go.

Geradores são funções que a cada interação retornam o próximo item de uma
sequência. Eles dão a possibilidade de se trabalhar um item da sequencia de
cada vez ao invés de ter que esperar pela sequencia completa para poder iterar
sobre os itens. Uma maneira de se fazer isso em go é usando channels e
goroutines e é isso que vamos fazer.


Uma primeira implementação
--------------------------

A ideia é criarmos uma função que retorna um channel que será alimentado por
uma goroutine e assim o consumidor da função poderá iterar sobre o channel.

Um exemplo clássico (e inútil) sempre usado é a sequencia de fibonacci, então
vamos fazer aqui também.

.. code-block:: go

   package main

   func Fibs() chan int {
       // Este canal será o consumido por quem usar esta função.
       ch := make(chan int)
       go func() {
	   // O Canal precisa ser fechado na função interna e não
	   // na externa porque senão o usuário não teria nada para
	   // consumir
	   defer close(ch)
	   i := 0
	   n := 1
	   for {
	       ch <- n
	       i, n = n, n+i
	   }
       }()
       return ch
   }

   func main() {
       for n := range Fibs() {
	   println(n)
       }
   }

Rodando este código vemos dois problemas. Um que o código não para nunca até
um C-c e quando acabam os números a sequencia para de crescer e começam a
aparecer uns números negativos.


Lidando com erros
-----------------

A primeira ideia para resolver isso é verifiar por erro e encerrar a goroutine
interior.

.. code-block:: go

   package main

   func Fibs() chan int {
       // Este canal será o consumido por quem usar esta função.
       ch := make(chan int)
       go func() {
	   // O Canal precisa ser fechado na função interna e não
	   // na externa porque senão o usuário não teria nada para
	   // consumir
	   defer close(ch)
	   i := 0
	   n := 1
	   for {
	       ch <- n
	       i, n = n, n+i
	       // Aqui se o número for negativo a gente termina,
	       // o canal será fechado e o consumidor vai sair do loop.
	       if n < 0 {
		   break
	       }
	   }
       }()
       return ch
   }

   func main() {
       for n := range Fibs() {
	   println(n)
       }
   }

Com isso o nosso programa já não fica rodando pra sempre e nem aparecem números
estranhos, mas o consumidor também não sabe o que aconteceu, a sequencia
simplesmente acabou. Se quisermos informar o consumidor sobre algum erro
ocorrido precisamos de um canal que carrega uma struct com o valor e um erro.

.. code-block:: go

   package main

   import "errors"

   type GenItem struct {
       Err error
       Val int
   }

   func Fibs() chan GenItem {
       // Este canal será o consumido por quem usar esta função.
       // Agora o canal é um canal de GenItem para conter também
       // informação sobre o erro
       ch := make(chan GenItem)
       go func() {
	   // O Canal precisa ser fechado na função interna e não
	   // na externa porque senão o usuário não teria nada para
	   // consumir
	   defer close(ch)
	   i := 0
	   n := 1
	   for {
	       item := GenItem{
		   Err: nil,
		   Val: n,
	       }
	       ch <- item
	       i, n = n, n+i
	       // Aqui se o número for negativo a gente termina
	       if n < 0 {
		   item := GenItem{
		       // A informação sobre o que de errado aconteceu
		       Err: errors.New("Cabou os número!"),
		       Val: 0,
		   }
		   ch <- item
		   break
	       }
	   }
       }()
       return ch
   }

   func main() {
       for item := range Fibs() {
	   if item.Err != nil {
	       panic(item.Err.Error())
	   }
	   println(item.Val)
       }
   }

Assim o consumidor tem toda a segurança pra entrar em pânico tranquilamente. :)


Botando uma galera pra trampar
------------------------------

Até agora vimos somente uma goroutine alimentando o canal. Vamos fazer um pouco
diferente dessa vez, vamos alimentar o canal com várias goroutines.

Imagine que temos uma lista de url e precisamos baixar o conteúdo de todas.
Podemos usar mais de uma goroutine para baixar o conteúdo concorrentemente e ir
alimentando o canal.

.. code-block:: go

    package main

    import (
	"io/ioutil"
	"net/http"
	"sync"
    )

    var URLS = []string{
	"https://tupi.poraodojuca.dev/index.html",
	"https://toxicbuild.poraodojuca.dev/index.html",
	"https://mongomotor.poraodojuca.dev/index.html",
    }

    // Alteramos GenItem para conter as iformações do download
    type GenItem struct {
	Err     error
	Content []byte
	Url     string
    }

    func DownloadUrls() chan GenItem {
	// Novamente o canal consumidor
	ch := make(chan GenItem)
	go func() {
	    // Note que o canal é fechado por esta goroutine
	    // e não pelas goroutines que fazem o downlaod as urls.
	    defer close(ch)

	    // Aqui usamos o WaitGroup para esperar até que todas as páginas
	    // tenham sido baixadas.
	    wg := new(sync.WaitGroup)
	    for _, url := range URLS {
		// aqui pra cada url a gente dispara uma goroutine e seque a vida
		// quem vai alimentar o canal é essa goroutina que baixa
		// a página.

		// Adicionamos 1 para cada goroutine que baixa uma página
		wg.Add(1)
		go func(url string) {
		    // Liberamos um do WaitGroup quando a função terminar
		    defer wg.Done()
		    content, err := DownloadUrl(url)
		    item := GenItem{
			Err:     err,
			Content: content,
			Url:     url,
		    }
		    ch <- item
		}(url)
	    }
	    // Aqui esperamos até que todas as páginas tenham sido baixadas
	    wg.Wait()
	}()
	return ch
    }

    // Isso aqui não importa, só faz um download normal mesmo
    func DownloadUrl(url string) ([]byte, error) {
	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
	    return nil, err
	}
	c := http.Client{}
	resp, err := c.Do(req)
	if err != nil {
	    return nil, err
	}

	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
	    return nil, err
	}
	return body, nil
    }

    func main() {
	for item := range DownloadUrls() {
	    if item.Err != nil {
		println("Erro baixando página " + item.Url)
	    } else {
		println("Página " + item.Url + " baixada com sucesso!")
	    }

	}
    }

Aqui tivemos bastantes coisas diferentes: Primeiro que as funções que alimentam
os canais são funções internas a função que cria o canal e também usamos um
WaitGroup para aguardar todas as goroutines produtoras terminarem. É preciso
esperar terminar porque caso contrário o canal seria fechado imediatamente.

Um wait group é um contador, para cada goroutine que queremos esperar adicionaos
1 ao wait group e o que estamos esperando deve remover um do grupo com Done()
quando terminar a execução. Wait() bloqueia a execução até que o contador do
wait group seja zerado.
Voltando pro começo

Até agora vimos como consumir um gerador até que os produtores terminem, mas e
se o consumidor quiser terminar antes? Se a gente simplesmente sair do loop o
canal vai ficar aberto eternamente, então  o que a gente precisa fazer é antes
de sair do loop fechar o canal e tratar no produtor a tentativa de escrever no
canal depois de fechado. Nosso primeiro exemplo fica assim:

.. code-block:: go

   package main

   import (
       "errors"
   )

   type GenItem struct {
       Err error
       Val int
   }

   func Fibs() chan GenItem {
       // Este canal será o consumido por quem usar esta função.
       // Agora o canal é um canal de GenItem para conter também
       // informação sobre o erro
       ch := make(chan GenItem)
       go func() {
	   defer func() {
	       // Quando o consumidor fechar o canal a gente vai tentar escrever
	       // num canal fechado. Chamando recover() a gente recupera o controle
	       // da execução.
	       if r := recover(); r != nil {
		   println("Recovering!")
	       }
	   }()
	   defer close(ch)
	   i := 0
	   n := 1
	   for {
	       item := GenItem{
		   Err: nil,
		   Val: n,
	       }
	       ch <- item
	       i, n = n, n+i
	       // Aqui se o número for negativo a gente termina
	       if n < 0 {
		   item := GenItem{
		       // A informação sobre o que de errado aconteceu
		       Err: errors.New("Cabou os número!"),
		       Val: 0,
		   }
		   ch <- item
		   break
	       }
	   }
       }()
       return ch
   }

   func main() {
       gen := Fibs()
       for item := range gen {
	   println(item.Val)
	   if item.Val > 1000 {
	       // Aqui precisamos fechar o canal antes de sair do loop
	       // os consumidores vão ter pânico quando tentarem escrever
	       // no canal fechado
	       close(gen)
	       break
	   }
       }
       println("Fim!")
   }

E acho que é isso.
