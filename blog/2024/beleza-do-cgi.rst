A beleza do Common Gateway Interface
====================================

.. post:: Oct 2, 2024
   :tags: computisses
   :author: Juca Crispim


Aqueles que já estão se aproximando da meia-idade vão se lembrar dos cgi scritps. Eles foram a primeira maneira de se fazer páginas dinâmicas por http, mas quando eu comecei a trabalhar como programador, cgi scripts já eram considerados ultrapassados, uma coisa que não se faz mais.


O fora de moda pode ser bom
---------------------------

Esses dias eu tava dando manutenção em um projetinho (não pergunta o porquê d'eu manter isso até hoje...) e queria subir pra vps nova. O problema era que eu usava o mod_perl do apache para servir via http. E aí eu precisaria instalar o apache que não uso pra nada dependências no meu código. Eu não tava nem um pouco feliz com isso, queria só rodar um script.

Foi aí que me lembrei do nosso bom e velho cgi. Aí implementei um plugin cgi para o tupi e é isso. Agora não preciso de nenhuma dependência, meu cgi script só precisa ler variáveis de ambiente, ler o stdin e o mandar o retorno pro stdout. Simples e bonito!


Nem tudo são flores
-------------------

Apesar de serem legais, cgi scripts caíram em desuso por alguns motivos. Cada chamada cria um novo processo, os scripts precisam todos parsear o corpo por si só e por aí vai. Mas pelo menos no caso de um processo por chamada, num tempo onde tem muita gente usando esses serverless que sobe uma insância nova só pra rodar uma função, o que é um processinho comparado à isso?
