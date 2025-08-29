Empacotamento e distribuição de projetos Python sem mistério - Adendo
=====================================================================

.. post:: Sep 15, 2015
   :tags: computisses
   :author: Juca Crispim

Oi, pessoal. Hoje o post é rapidinho, só um adendo ao post anterior sobre
empacotamento de projetos Python.

A coisa é a seguinte: A maneira recomendada para fazer o upload do pacote para
o pypi agora é usando o twine. Esse carinha é o recomendado agora porque,
diferentemente do pip, ele faz o upload usando https.

Então, quando formos empacotar e distribuir nosso programa, ao invés de usarmos:

.. code-block::

   $ python setup.py sdist upload


Para criar o pacote e subi-lo, usaremos o comando sdist para criar o pacote e
depois usaremos o twine para fazer o upload, assim:

.. code-block::

   $ python setup.py sdist
   $ twine upload dist/my-package-0.1.tar.gz

E é isso!
