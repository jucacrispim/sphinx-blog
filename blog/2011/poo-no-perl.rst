.. post:: Dec 28, 2011
   :tags: computisses
   :author: Juca Crispim


Poo no Perl!
============

Boas pessoal. Tô de volta. :) Depois de um bom tempo sem postar nada, vou
aproveitar minha semaninha de folga pra tirar as teias do porão e vamos falar
um pouquinho sobre orientação a objetos no Perl.


Mas o Perl é um cara sem classe, não?
-------------------------------------

No Perl realmente não existe uma palavra mágica class ou algo do tipo, mas a
gente pode criar classes sim!

As classes no Perl são criadas o mecanismo de package que também é usado mais
usualmente para criar módulos tradicionais. E os objetos... bom, os objetos são
referências abençoadas ! :P


Referência abençoada? Que diabo é isso?
---------------------------------------

Calma... nada de misticismos. Uma referência abençoada é só uma referência que
foi passada como argumento para a função bless. É essa função a responsável por
dizer: 'essa referência pertence a este package (classe)'. Depois de abençoada,
você será capaz de chamar funções através dessa referência, ou, em outras
palavras, você terá métodos!


Entendi, mas já cansei de papo...
---------------------------------

Beleza, já falei bastente, e agora é hora do código.

Pra este exemplo vou criar uma classe (package) Pessoa (como um módulo normal
do Perl, isso será gravado em um arquivo chamado Pessoa.pm), e nessa classe
criarei o contrutor da classe e os seus outros métodos. Agora sim, o código:

.. code-block:: perl

   package Pessoa; # Arquivo Pessoa.pm

   use strict;
   use warnings;

   # Aqui é o contrutor da classe.
   # O nome 'new' não é uma obrigação, só
   # uma convenção de uso.
   sub new{
       # classe onde o a referência será abençoada.
       # Isso aqui não precisa ser passado como parâmetro
       # na hora de instanciar o objeto, é passado implicitamente
       my $class = shift;

       # Nossa referência que será abençoada.
       # Pode ser uma referência qualquer, não necessáriamente
       # uma referência a um hash;
       my $self = {};
       $self->{NOME} = undef;
       $self->{IDADE} = undef;

       # Agora a mágica acontece
       bless($self, $class);

       return $self;
   }

   # Métodos pra proteger o acesso aos nossos atributos
   sub nome{
       # Como $class no contrutor, $self aqui também é parâmetro implícito
       my $self = shift;
       $self->{NOME} = shift if @_;
       return $self->{NOME};
   }

   sub idade{
       my $self = shift;
       $self->{IDADE} = shift if @_;
       return $self->{IDADE};
   }

   # Outro metodozinho só de exemplo
   sub fale{
       my $self = shift;
       my $fala = shift || "Qualquer coisa";
       return $fala
   }

   # Precisa retornar um valor verdadeiro
   1;

Agora, vamos criar uma subclasse de Pessoa

.. code-block:: perl

   package PessoaMuda; # Arquivo PessoaMuda.pm

   # Importando a super classe
   use Pessoa;

   # Aqui estamos dizendo que PessoaMuda é uma Pessoa
   @ISA = (Pessoa);

   # Sobrescrevendo o método fale pra que retorne nada...
   sub fale{
       my $self = shift;
       return
   }

   1;

E, por fim, vamos usar nossos objetos

.. code-block:: perl

   #!/usr/bin/env perl

   use strict;
   use warnings;

   # Importando nossas classes
   use Pessoa;
   use PessoaMuda;

   # Nova instância de Pessoa;
   my $p = Pessoa->new();
   $p->nome('Alguém');
   print $p->nome();

   $p->idade(10000);
   print $p->idade();

   print $p->fale('Eu nasci a 10000 anos atrás...') . "\n";

   # Uma pessoa muda
   my $pm = PessoaMuda->new();
   $pm->nome('Ninguém');
   print $pm->nome();

   $pm->idade(10000);
   print $pm->idade();

   # Mudos não falam, lembra? Sobrescrevemos o método...
   print $pm->fale('Alguma coisa');

Bom, é isso! Pra mais sobre orientação a objetos no Perl, leiam o perltoot.

Valeu, e até a próxima!
