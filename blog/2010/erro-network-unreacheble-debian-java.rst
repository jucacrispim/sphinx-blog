.. post:: Mar 4, 2010
   :tags: computisses
   :author: Juca Crispim

Corrigir erro 'Network Unreachable' com Java no Debian
======================================================

Puta, perdi uma tarde inteira quebrando a cabeça com isso. Estava tentando usar
o Squirrel-SQL (e o Sql Developer também), mas sempre dava merda, com o java
dizendo que "Network Unreachble". Depois de bastante tempo, esse cara me deu a
resposta. O que acontece é que no arquivo /etc/sysctl.d/bindv6only.conf a
variável net.ipv6.bindv6only estava marcada como 1, e essa é a fonte do
problema. Esta variável tem que estar marcada como 0. Pra arrumar isso, use:

.. code-block:: sh

   # sed -i 's/net.ipv6.bindv6only\ =\ 1/net.ipv6.bindv6only\ =\ 0/' /etc/sysctl.d/bindv6only.conf
   # invoke-rc.d procps restart

É isso aí.
