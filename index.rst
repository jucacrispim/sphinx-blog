
.. Porão do Juca index file, created by `ablog start` on Fri Nov  1 19:36:43 2024.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.



Porão do Juca
=============

.. postlist:: 5
   :excerpts:
   :format: {title}
   :expand: Leia mais...

`Ver todos os posts <blog/>`_

.. `toctree` directive, below, contains list of non-post `.rst` files.
   This is how they appear in Navigation sidebar. Note that directive
   also contains `:hidden:` option so that it is not included inside the page.

   Posts are excluded from this directive so that they aren't double listed
   in the sidebar both under Navigation and Recent Posts.

.. photolist:: 5

`Ver todas as fotos <fotos/>`_

.. toctree::
   :hidden:

   blog/index.rst
   fotos/index.rst
   Sobre <sobre.rst>
   feed.rst
