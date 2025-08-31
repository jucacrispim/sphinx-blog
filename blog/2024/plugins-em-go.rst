Plugins in go (lang, não horse)
===============================

.. post:: Jul 5, 2024
   :category: computisses
   :author: Juca Crispim


Eu gosto bastante da ideia de programas que potencialmente podem ser a junção e peças menores, unix pipes, serviços remotos e coisas do tipo e permitem escrever programas menores que se comunicam via uma interface definida. Nessa mesma linha, plugins podem ser bem úteis extendendo um programa sem que o programa principal saiba do que está acontecendo.

Em go existe o package plugin que permite escrever e carregar plugins dinamicamente. Basicamente o que precisa ser feito é definir o que o plugin precisa implementar (uma ou mais funções) e o programa principal carrega o plugin em tempo de execução e executa as funções implementadas pelo plugin.


O programa principal
--------------------

A primeira coisa que precisamos é de um programa que seja capaz de carregar e executar plugins. Este será o programa principal que será extendidos pelos plugins. Para carregar o plugin se usa plugin.Open e  plugin.Plugin.Lookup.

Então comecemos com um programa bem simples, sem suporte a plugins:

.. code-block:: go

   package main

   import (
       "flag"
       "os"
   )

   func main() {
       action := flag.String("action", "faz", "")
       flag.CommandLine.Parse(os.Args[1:])

       s := "coisa"

       println(*action + " " + s)
   }

.. code-block:: sh

   $ ./programa
   faz coisa
   $./programa -action outra
   outra coisa


Nada muito o que explicar aqui, né? Então agora vamos implementar o suporte a plugins.

A ideia agora é que a gente vai usar plugins baseados no parâmetro action que o
usuário passar.

.. code-block:: go

   package main

   import (
       "flag"
       "fmt"
       "os"
   v    "plugin"
   )

   func main() {
       action := flag.String("action", "faz", "")
       flag.CommandLine.Parse(os.Args[1:])
       s := "coisa"

       var f plugin.Symbol
       var fn func(string)
       var ok bool

       // Aqui plugin.Open recebe o caminho do plugin.
       // A gente vai procurar por um plugin baseado
       // na action que o usuário passou
       fpath := fmt.Sprintf("./%s_plugin.so", *action)
       p, err := plugin.Open(fpath)

       if err != nil {
	   // Olha esse goto maroto engolindo os bug tudo!
	   goto PRINT
       }

       // Aqui a gente procura no plugin por um símbolo
       // com o nome de Action
       f, err = p.Lookup("Action")
       if err != nil {
	   goto PRINT
       }

       // Aqui verifica se o símbolo é realmente do tipo
       // que a gente espera
       fn, ok = f.(func(string))
       if !ok {
	   goto PRINT
       }

       fn(s)
       return

   PRINT:
       println(*action + " " + s)

   }


Por enquanto como não temos nenhum plugin nosso programa continua fazendo a mesma coisa:

.. code-block::

   $./programa
   faz coisa
   $./programa -action outra
   outra coisa

O plugin

Para implementar o plugin a gente precisa só implementar uma função chamada Action que
recebe uma string como parâmetro:

.. code-block:: go

   package main

   func Action(s string) {
       println("Aqui o plugin fazendo outra " + s)
   }

   Agora o plugin precisa ser compilado com a flag -buildmode=plugin

   go build -buildmode=plugin outra_plugin.go

   E agora nosso programa já pode usar o plugin:


.. code-block:: sh

   $./programa
   faz coisa
   $./programa -action outra
   Aqui o plugin fazendo outra coisa

E é isso!
