.. post:: Dec 10, 2025
   :category: computisses
   :author: Juca Crispim


.. _internacionalizacao-go:

Internacionalizando e localizando projetos em go
================================================

Internacionalização é o que a gente faz pra que nossos programas sejam
capazes de exibir as informações em diferentes idiomas, moedas,
usando diferentes formatos de numeração, datas, diferentes calendários,
enfim, todas as informações que  um usuário vê. Já a localização é o processo
específico de usar as ferramentas do programa para adaptar as coisas para
uma determinada localidade.


A ideia
-------

Para o exemplo eu precisava de algo que eu pudesse mostrar além de idioma,
formatos de data e números. Depois de não ter ideia boa nenhuma eu resolvi
ir com uma ideia ruim mesmo: uma aplicação que mostra informações de cidades. Uau!
É besta, mas pelo menos tem números, datas e um textinho pra traduzir.

Então, uma primeira ideia seria algo assim:

.. code-block:: go

   package main

   import (
	   "fmt"
	   "time"
   )

   type CityInfo struct {
	   Name       string
	   Country    string
	   Population int
	   // Em quilômetros quadrados
	   Area float64
	   // Data de fundação da cidade
	   Foundation time.Time
   }

   var allCities = []CityInfo{
	   {
		   Name:       "São Paulo",
		   Population: 11450000,
		   Area:       1521,
		   Foundation: time.Date(1554, 1, 25, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       "New York",
		   Population: 8500000,
		   Area:       1213,
		   Foundation: time.Date(1609, 9, 3, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       "Buenos Aires",
		   Population: 2891082,
		   Area:       202,
		   Foundation: time.Date(1536, 2, 3, 0, 0, 0, 0, time.UTC),
	   },
   }

   func main() {
	   for _, city := range allCities {
		   msg := fmt.Sprintf(
			   "The city of %s was founded in %s. It has an area of %.2f squared kilometers and a population of %d people\n",
			   city.Name, city.Foundation.Format("2006-01-02"), city.Area, city.Population)
		   println(msg)

	   }
   }


Rodando isso a gente tem o seguinte:

.. code-block:: text

   $ go build cityinfo.go
   $ ./cityinfo
   The city of São Paulo was founded in 1554-01-25. It has an area of 1521.00 squared kilometers and a population of 11450000 people

   The city of New York was founded in 1609-09-03. It has an area of 1213.00 squared kilometers and a population of 8500000 people

   The city of Buenos Aires was founded in 1536-02-03. It has an area of 202.00 squared kilometers and a population of 2891082 people

A ideia aí é que a base seja feita em inglês, que é a língua franca atual, a formatação de data está como
yyyy-mm-dd porque é o que menos pode dar confusão, a unidade para distância é em quilômetros porque
é a medida do sistema internacional e o número está sem formatação nenhuma. Basicamente a base
deveria sempre ser algo o mais 'neutro' possível.

Agora vamos internacionalizar a coisa pra que possamos ter traduções e formatações específicas
por localidade.


Internacionalizando
-------------------

Para internacionalizar, a gente precisa refatorar um pouco o código e deixar as coisas
preparadas. O que a gente precisa localizar aí é o idioma, a formatação de número,
formatação de data e a únidade de medida usada.

Refatorei o código usando o `gotext <https://github.com/leonelquinteros/gotext>`_
e ficou assim:

Agora a gente pode refatorar o nosso código. Ficou assim:

.. code-block:: go

   package main

   import (
	   "bytes"
	   "embed"
	   "fmt"
	   "html/template"
	   "os"
	   "strings"
	   "time"

	   "github.com/leonelquinteros/gotext"
   )

   // o diretório onde ficarão as traduções
   const localesDir = "locales"
   const defaultDomain = "default"

   //go:embed locales
   var embeddedLocales embed.FS

   var locales = make(map[string]func() Locale)

   var DEFAULT_DTFMT = "2006-01-02"

   // Uma iterface para a localzação. Cada localicação
   // será uma implementação dessa interface
   type Locale interface {
	   // o gotext.Locale que é o responsável pela tradução
	   // dos textos mostrados ao usuário
	   Text() *gotext.Locale
	   // As funções Format* a gente usa pra formatar coisas
	   // de maneira diferente pra casa localização
	   FormatDate(time.Time) string
	   FormatInt(int) string
	   FormatFloat64(float64) string
	   // A unide de medida também pode mudar por localização
	   GetAreaUnit() string
	   // Faz a conversão do valor em quilômetros (o padrão)
	   // para a unidade usada na localização
	   NormalizeArea(float64) float64
   }

   // Uma versãod de printf que usa o mecanismo de template
   // do go assim a gente pode usar os argumentos como chave/valor
   // ao invés de argumentos posicionais. Isso é útil porque
   // a ordem das palavras pode mudar de acordo com o idioma.
   func Tprintf(tmpl string, data map[string]any) string {
	   t := template.Must(template.New("translation").Parse(tmpl))
	   buf := &bytes.Buffer{}
	   if err := t.Execute(buf, data); err != nil {
		   // notest
		   return tmpl
	   }
	   return buf.String()
   }

   // O locale padrão que será usado quando a localização
   // para o usuário não estiver disponível
   type DefaultLocale struct {
	   l     *gotext.Locale
	   dtfmt string
   }

   func (loc DefaultLocale) Text() *gotext.Locale {
	   return loc.l
   }

   func (loc DefaultLocale) FormatDate(dt time.Time) string {
	   return dt.Format(loc.dtfmt)
   }

   func (loc DefaultLocale) FormatInt(n int) string {
	   return fmt.Sprintf("%d", n)
   }

   func (loc DefaultLocale) FormatFloat64(n float64) string {
	   return fmt.Sprintf("%.2f", n)
   }

   func (loc DefaultLocale) GetAreaUnit() string {
	   l := loc.Text()
	   return l.Get("kilometers")
   }

   func (loc DefaultLocale) NormalizeArea(n float64) float64 {
	   return n
   }

   func NewDefaultLocale() Locale {
	   lang := "C"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   loc := DefaultLocale{l: l, dtfmt: DEFAULT_DTFMT}
	   return loc
   }

   func RegisterLocale(label string, fn func() Locale) {
	   locales[label] = fn
   }


   func RegisterAllLocales() {
   }

   func GetLocale() Locale {
	   RegisterAllLocales()
	   // a gente pega o idioma padrão do sistema
	   lang := os.Getenv("LANG")
	   lang = strings.Split(lang, ".")[0]

	   fn, exists := locales[lang]
	   if !exists {
		   fn = NewDefaultLocale
	   }
	   return fn()
   }

   type CityInfo struct {
	   Name       string
	   Country    string
	   Population int
	   // Em quilômetros quadrados
	   Area float64
	   // Data de fundação da cidade
	   Foundation time.Time
   }

   var loc Locale = GetLocale()
   var locText *gotext.Locale = loc.Text()

   var allCities = []CityInfo{
	   {
		   // A função Get de gotext.Locale marca um texto como traduzível.
		   // As strings passadas pra essa função serão extraídas pelo xgotext
		   Name:       locText.Get("São Paulo"),
		   Population: 11450000,
		   Area:       1521,
		   Foundation: time.Date(1554, 1, 25, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       locText.Get("New York"),
		   Population: 8500000,
		   Area:       1213,
		   Foundation: time.Date(1609, 9, 3, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       locText.Get("Buenos Aires"),
		   Population: 2891082,
		   Area:       202,
		   Foundation: time.Date(1536, 2, 3, 0, 0, 0, 0, time.UTC),
	   },
   }

   func main() {
	   for _, city := range allCities {
		   name := city.Name
		   foundation := loc.FormatDate(city.Foundation)
		   area := loc.FormatFloat64(loc.NormalizeArea(city.Area))
		   pop := loc.FormatInt(city.Population)
		   areaUnit := loc.GetAreaUnit()
		   fmtArgs := make(map[string]any)
		   fmtArgs["name"] = name
		   fmtArgs["foundation"] = foundation
		   fmtArgs["area"] = area
		   fmtArgs["areaUnit"] = areaUnit
		   fmtArgs["pop"] = pop
		   msg := Tprintf(
			   locText.Get(
				   "The city of {{.name}} was founded in {{.foundation}}. It has an area of {{.area}} squared {{.areaUnit}} and a population of {{.pop}} people\n"), fmtArgs)
		   println(msg)

	   }
   }


Agora, antes de compilar isso, a gente precisa criar o diretório onde
ficarão as traduções e extrair as strings a serem extraídas. Assim:

.. code-block:: text

   $ mkdir locales
   $ xgotext -in . -out locales -default messages


Agora a gente pode compilar e rodar o programa:

.. code-block:: text

   $ go build cityinfo.go
   $ ./cityinfo
   The city of São Paulo was founded in 1554-01-25. It has an area of 1521.00 squared kilometers and a population of 11450000 people

   The city of New York was founded in 1609-09-03. It has an area of 1213.00 squared kilometers and a population of 8500000 people

   The city of Buenos Aires was founded in 1536-02-03. It has an area of 202.00 squared kilometers and a population of 2891082 people


A saida ainda é a mesma, mas agora a gente pode adicionar mais localizações ao nosso
programa.


.. note::

   A ideia aqui de usar uma interace e uma implementação da interface
   pra cada locale é que, apesar de ser possível fazer de uma outra
   maneira, um pouco mais 'dinâmica', locales são tão variados que
   no fim fica mais fácil ter locales diferentes assim cada locale
   sabe o que precisa para a localicação especifica


Localizando
-----------

Agora a gente pode fazer as implementações específicas para cada
localidade que a gente vai suportar. A gente vai usar
`x/text/language <https://pkg.go.dev/golang.org/x/text/language>`_ e
`x/text/message <https://pkg.go.dev/golang.org/x/text/message>`_
para formatar os números.

A implementação dos locales para formatação/unidades de medida
ficou assim:

.. code-block:: go

   // Locale para português do Brasil
   type PTBRLocale struct {
	   DefaultLocale
   }

   func (loc PTBRLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.BrazilianPortuguese)
	   return printer.Sprintf("%d", n)
   }

   func (loc PTBRLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.BrazilianPortuguese)
	   return printer.Sprintf("%.2f", n)
   }

   func NewPTBRLocale() Locale {
	   lang := "pt_BR"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: DDMMYYYY_DTFMT}
	   loc := PTBRLocale{DefaultLocale: defloc}
	   return loc
   }

   // Locale para espanhol
   type ESARLocale struct {
	   DefaultLocale
   }

   func (loc ESARLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.LatinAmericanSpanish)
	   return printer.Sprintf("%d", n)
   }

   func (loc ESARLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.LatinAmericanSpanish)
	   return printer.Sprintf("%.2f", n)
   }

   func NewESARLocale() Locale {
	   lang := "es_AR"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: DDMMYYYY_DTFMT}
	   loc := ESARLocale{DefaultLocale: defloc}
	   return loc
   }

   // Locale para inglês dos eua
   type ENUSLocale struct {
	   DefaultLocale
   }

   func (loc ENUSLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.AmericanEnglish)
	   return printer.Sprintf("%d", n)
   }

   func (loc ENUSLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.AmericanEnglish)
	   return printer.Sprintf("%.2f", n)
   }

   // Para inglês dos eua vamos mostrar em milhas
   func (loc ENUSLocale) GetAreaUnit() string {
	   l := loc.Text()
	   return l.Get("miles")
   }

   // Transforma a área de km quadrados para milhas
   func (loc ENUSLocale) NormalizeArea(n float64) float64 {
	   return n / 0.386102
   }

   func NewENUSLocale() Locale {
	   lang := "en_US"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: MMDDYYYY_DTFMT}
	   loc := ENUSLocale{DefaultLocale: defloc}
	   return loc
   }


Alterei também a função ``RegisterAllLocales`` para registrar os locales
que criamos. Assim:

.. code-block:: go

   func RegisterAllLocales() {
	   RegisterLocale("pt_BR", NewPTBRLocale)
	   RegisterLocale("es_AR", NewESARLocale)
	   RegisterLocale("en_US", NewENUSLocale)
   }

Agora a gente precisa só traduzir os textos mostrados ao usuário. Primeiro
a gente cria um diretório onde vão ficar as traduções do idioma assim:

.. code-block:: sh

   $ mkdir -p locales/pt_BR/LC_MESSAGES


Agora copiamos o arquivo que foi gerado quando extraímos as strings traduzíveis
para o diretório do idioma

.. code-block:: sh

   $ cp locales/messages.pot locales/pt_BR/LC_MESSAGES/default.po


.. note::

   Essa cópia do arquivo de mensagens só se faz na primeira vez,
   das próximas quando forem atualizados textos no programa, é
   só extrair as strings novamente com o comando ``xgotext``
   e usar o comando ``msgmerge`` para atualizar o arquivo de
   traduções. Assim:

   .. code-block:: sh

      $ msgmerge -U locales/pt_BR/LC_MESSAGES/default.po locales/messages.pot

O arquivo com as traduções é algo assim:

.. code-block:: text

   msgid ""
   msgstr ""
   "Language: \n"
   "MIME-Version: 1.0\n"
   "Content-Type: text/plain; charset=UTF-8\n"
   "Content-Transfer-Encoding: 8bit\n"
   "Plural-Forms: nplurals=2; plural=(n != 1);\n"
   "X-Generator: xgotext\n"

   #: cityinfo.go:236
   msgid "Buenos Aires"
   msgstr ""

   #: cityinfo.go:230
   msgid "New York"
   msgstr ""

   #: cityinfo.go:224
   msgid "São Paulo"
   msgstr ""

   #: cityinfo.go:257
   msgid ""
   "The city of {{.name}} was founded in {{.foundation}}. It has an area of "
   "{{.area}} squared {{.areaUnit}} and a population of {{.pop}} people\n"
   msgstr ""

   #: cityinfo.go:86
   msgid "kilometers"
   msgstr ""

   #: cityinfo.go:167
   msgid "miles"
   msgstr ""


Agora precisa traduzir as strings, ficando assim:

.. code-block:: text

   msgid ""
   msgstr ""
   "Language: \n"
   "MIME-Version: 1.0\n"
   "Content-Type: text/plain; charset=UTF-8\n"
   "Content-Transfer-Encoding: 8bit\n"
   "Plural-Forms: nplurals=2; plural=(n != 1);\n"
   "X-Generator: xgotext\n"

   #: cityinfo.go:236
   msgid "Buenos Aires"
   msgstr ""

   #: cityinfo.go:230
   msgid "New York"
   msgstr "Nova Iorque"

   #: cityinfo.go:224
   msgid "São Paulo"
   msgstr ""

   #: cityinfo.go:257
   msgid ""
   "The city of {{.name}} was founded in {{.foundation}}. It has an area of "
   "{{.area}} squared {{.areaUnit}} and a population of {{.pop}} people\n"
   msgstr ""
   "A cidade de {{.name}} foi fundada em {{.foundation}}. Tem uma área de "
   "{{.area}} {{.areaUnit}} quadrados e uma população de {{.pop}} de pessoas\n"

   #: cityinfo.go:86
   msgid "kilometers"
   msgstr "quilômetros"

   #: cityinfo.go:167
   msgid "miles"
   msgstr "milhas"

E aí precisa repetir esse mesmo procediemnto para todos os locales suportados.

Depois das localizações e traduções é só rodar nosso programa:

.. code-block:: text

   $ go build cityinfo.go
   $ ./cityinfo
   A cidade de São Paulo foi fundada em 25/01/1554. Tem uma área de 1.521,00 quilômetros quadrados e uma população de 11.450.000 de pessoas

   A cidade de Nova Iorque foi fundada em 03/09/1609. Tem uma área de 1.213,00 quilômetros quadrados e uma população de 8.500.000 de pessoas

   A cidade de Buenos Aires foi fundada em 03/02/1536. Tem uma área de 202,00 quilômetros quadrados e uma população de 2.891.082 de pessoas


Como o idioma padrão do meu sistema é português do Brasil foi o locale usado pelo programa,
mas a gente pode alterar isso usando a variável de ambiente ``LANG``

.. code-block:: text

   $ LANG=es_AR;./cityinfo
   La ciudad de San Pablo fue fundada en 25/01/1554. Tiene una área de 1,521.00 kilómetros quadrados y una población de 11,450,000 de personas

   La ciudad de Nueva York fue fundada en 03/09/1609. Tiene una área de 1,213.00 kilómetros quadrados y una población de 8,500,000 de personas

   La ciudad de Buenos Aires fue fundada en 03/02/1536. Tiene una área de 202.00 kilómetros quadrados y una población de 2,891,082 de personas

   $ LANG=en_US;./cityinfo
   The city of São Paulo was founded in 01/25/1554. It has an area of 3,939.37 squared miles and a population of 11,450,000 people

   The city of New York was founded in 09/03/1609. It has an area of 3,141.66 squared miles and a population of 8,500,000 people

   The city of Buenos Aires was founded in 02/03/1536. It has an area of 523.18 squared miles and a population of 2,891,082 people

O código completo ficou assim:

.. code-block:: go

   package main

   import (
	   "bytes"
	   "embed"
	   "fmt"
	   "html/template"
	   "os"
	   "strings"
	   "time"

	   "github.com/leonelquinteros/gotext"
	   "golang.org/x/text/language"
	   "golang.org/x/text/message"
   )

   // o diretório onde ficarão as traduções
   const localesDir = "locales"
   const defaultDomain = "default"

   //go:embed locales
   var embeddedLocales embed.FS

   var locales = make(map[string]func() Locale)

   var DEFAULT_DTFMT = "2006-01-02"
   var DDMMYYYY_DTFMT = "02/01/2006"
   var MMDDYYYY_DTFMT = "01/02/2006"

   // Uma iterface para a localzação. Cada localicação
   // será uma implementação dessa interface
   type Locale interface {
	   // o gotext.Locale que é o responsável pela tradução
	   // dos textos mostrados ao usuário
	   Text() *gotext.Locale
	   // As funções Format* a gente usa pra formatar coisas
	   // de maneira diferente pra casa localização
	   FormatDate(time.Time) string
	   FormatInt(int) string
	   FormatFloat64(float64) string
	   // A unide de medida também pode mudar por localização
	   GetAreaUnit() string
	   // Faz a conversão do valor em quilômetros (o padrão)
	   // para a unidade usada na localização
	   NormalizeArea(float64) float64
   }

   // Uma versãod de printf que usa o mecanismo de template
   // do go assim a gente pode usar os argumentos como chave/valor
   // ao invés de argumentos posicionais. Isso é útil porque
   // a ordem das palavras pode mudar de acordo com o idioma.
   func Tprintf(tmpl string, data map[string]any) string {
	   t := template.Must(template.New("translation").Parse(tmpl))
	   buf := &bytes.Buffer{}
	   if err := t.Execute(buf, data); err != nil {
		   // notest
		   return tmpl
	   }
	   return buf.String()
   }

   // O locale padrão que será usado quando a localização
   // para o usuário não estiver disponível
   type DefaultLocale struct {
	   l     *gotext.Locale
	   dtfmt string
   }

   func (loc DefaultLocale) Text() *gotext.Locale {
	   return loc.l
   }

   func (loc DefaultLocale) FormatDate(dt time.Time) string {
	   return dt.Format(loc.dtfmt)
   }

   func (loc DefaultLocale) FormatInt(n int) string {
	   return fmt.Sprintf("%d", n)
   }

   func (loc DefaultLocale) FormatFloat64(n float64) string {
	   return fmt.Sprintf("%.2f", n)
   }

   func (loc DefaultLocale) GetAreaUnit() string {
	   l := loc.Text()
	   return l.Get("kilometers")
   }

   func (loc DefaultLocale) NormalizeArea(n float64) float64 {
	   return n
   }

   func NewDefaultLocale() Locale {
	   lang := "C"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   loc := DefaultLocale{l: l, dtfmt: DEFAULT_DTFMT}
	   return loc
   }

   // Locale para português do Brasil
   type PTBRLocale struct {
	   DefaultLocale
   }

   func (loc PTBRLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.BrazilianPortuguese)
	   return printer.Sprintf("%d", n)
   }

   func (loc PTBRLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.BrazilianPortuguese)
	   return printer.Sprintf("%.2f", n)
   }

   func NewPTBRLocale() Locale {
	   lang := "pt_BR"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: DDMMYYYY_DTFMT}
	   loc := PTBRLocale{DefaultLocale: defloc}
	   return loc
   }

   // Locale para espanhol
   type ESARLocale struct {
	   DefaultLocale
   }

   func (loc ESARLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.LatinAmericanSpanish)
	   return printer.Sprintf("%d", n)
   }

   func (loc ESARLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.LatinAmericanSpanish)
	   return printer.Sprintf("%.2f", n)
   }

   func NewESARLocale() Locale {
	   lang := "es_AR"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: DDMMYYYY_DTFMT}
	   loc := ESARLocale{DefaultLocale: defloc}
	   return loc
   }

   // Locale para inglês dos eua
   type ENUSLocale struct {
	   DefaultLocale
   }

   func (loc ENUSLocale) FormatInt(n int) string {
	   printer := message.NewPrinter(language.AmericanEnglish)
	   return printer.Sprintf("%d", n)
   }

   func (loc ENUSLocale) FormatFloat64(n float64) string {
	   printer := message.NewPrinter(language.AmericanEnglish)
	   return printer.Sprintf("%.2f", n)
   }

   // Para inglês dos eua vamos mostrar em milhas
   func (loc ENUSLocale) GetAreaUnit() string {
	   l := loc.Text()
	   return l.Get("miles")
   }

   // Transforma a área de km quadrados para milhas
   func (loc ENUSLocale) NormalizeArea(n float64) float64 {
	   return n / 0.386102
   }

   func NewENUSLocale() Locale {
	   lang := "en_US"
	   l := gotext.NewLocaleFSWithPath(lang, embeddedLocales, localesDir)
	   l.AddDomain(defaultDomain)
	   defloc := DefaultLocale{l: l, dtfmt: MMDDYYYY_DTFMT}
	   loc := ENUSLocale{DefaultLocale: defloc}
	   return loc
   }

   func RegisterLocale(label string, fn func() Locale) {
	   locales[label] = fn
   }

   func RegisterAllLocales() {
	   RegisterLocale("pt_BR", NewPTBRLocale)
	   RegisterLocale("es_AR", NewESARLocale)
	   RegisterLocale("en_US", NewENUSLocale)
   }

   func GetLocale() Locale {
	   RegisterAllLocales()
	   // a gente pega o idioma padrão do sistema
	   lang := os.Getenv("LANG")
	   lang = strings.Split(lang, ".")[0]

	   fn, exists := locales[lang]
	   if !exists {
		   fn = NewDefaultLocale
	   }
	   return fn()
   }

   type CityInfo struct {
	   Name       string
	   Country    string
	   Population int
	   // Em quilômetros quadrados
	   Area float64
	   // Data de fundação da cidade
	   Foundation time.Time
   }

   var loc Locale = GetLocale()
   var locText *gotext.Locale = loc.Text()

   var allCities = []CityInfo{
	   {
		   // A função Get de gotext.Locale marca um texto como traduzível.
		   // As strings passadas pra essa função serão extraídas pelo xgotext
		   Name:       locText.Get("São Paulo"),
		   Population: 11450000,
		   Area:       1521,
		   Foundation: time.Date(1554, 1, 25, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       locText.Get("New York"),
		   Population: 8500000,
		   Area:       1213,
		   Foundation: time.Date(1609, 9, 3, 0, 0, 0, 0, time.UTC),
	   },
	   {
		   Name:       locText.Get("Buenos Aires"),
		   Population: 2891082,
		   Area:       202,
		   Foundation: time.Date(1536, 2, 3, 0, 0, 0, 0, time.UTC),
	   },
   }

   func main() {
	   for _, city := range allCities {
		   name := city.Name
		   foundation := loc.FormatDate(city.Foundation)
		   area := loc.FormatFloat64(loc.NormalizeArea(city.Area))
		   pop := loc.FormatInt(city.Population)
		   areaUnit := loc.GetAreaUnit()
		   fmtArgs := make(map[string]any)
		   fmtArgs["name"] = name
		   fmtArgs["foundation"] = foundation
		   fmtArgs["area"] = area
		   fmtArgs["areaUnit"] = areaUnit
		   fmtArgs["pop"] = pop
		   msg := Tprintf(
			   locText.Get(
				   "The city of {{.name}} was founded in {{.foundation}}. It has an area of {{.area}} squared {{.areaUnit}} and a population of {{.pop}} people\n"), fmtArgs)
		   println(msg)

	   }
   }

É isso!
