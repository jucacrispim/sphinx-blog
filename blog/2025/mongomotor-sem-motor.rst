.. post:: Sep 29, 2025
   :category: computisses
   :author: Juca Crispim


Mongomotor sem motor
====================

O `mongomotor <https://mongomotor.poraodojuca.dev>`_ é um ORM assíncrono
para mongodb baseado no `mongoengine <http://mongoengine.org>`_ e que
usava o motor como *driver* assíncrono. Mas agora o motor foi *deprecated*
em favor do pymongo assíncrono, então eu precisava mexer no mongomotor
para trocar o driver. Com isso surgiu a terceira encarnação do mongomotor,
que é meio que do mesmo jeito que a primeira.

Como era?
=========

Do lado do motor/pymongo o que mudou foi que o motor usa um *pool* de *threads*
para rodar as operações de *IO*, já o pymongo assíncrono implementa o suporte
à asyncio diretamente no pymongo. A api, em teoria, é a mesma do motor. Já do
meu lado, no mongomotor, eu usava toda a estrutura do motor pra também rodar
as operações num *pool* de *threads*, e como manter meu próprio *fork* do
motor não era de jeito nenhum uma opção, eu precisaria mudar como o mongomotor
funciona.

Esses dois módulos aqui (que já não existem mais) eram o cerne da coisa:

.. code-block:: python

   # metaprogramming.py

   import functools
   from mongoengine.base.metaclasses import (
       TopLevelDocumentMetaclass,
       DocumentMetaclass
   )
   from mongoengine.connection import DEFAULT_CONNECTION_NAME
   from mongoengine.context_managers import switch_db as me_switch_db
   from motor.metaprogramming import MotorAttributeFactory
   from pymongo.database import Database
   from mongomotor import utils
   from mongomotor.exceptions import ConfusionError
   from mongomotor.monkey import MonkeyPatcher


   def asynchronize(method, cls_meth=False):
       """Decorates `method` so it returns a Future.

       :param method: A mongoengine method to be asynchronized.
       :param cls_meth: Indicates if the method being asynchronized is
	  a class method."""

       # If we are not in the main thread, things are already asynchronized
       # so we don't need to asynchronize it again.
       if not utils.is_main_thread():
	   return method

       def async_method(instance_or_class, *args, **kwargs):
	   framework = get_framework(instance_or_class)
	   loop = kwargs.pop('_loop', None) or get_loop(instance_or_class)

	   future = framework.run_on_executor(
	       loop, method, instance_or_class, *args, **kwargs)
	   return future

       # for mm_extensions.py (docs)
       async_method.is_async_method = True
       async_method = functools.wraps(method)(async_method)
       if cls_meth:
	   async_method = classmethod(async_method)

       return async_method


   def synchronize(method, cls_meth=False):
       """Runs method while using the synchronous pymongo driver.

       :param method: A mongoengine method to run using the pymongo driver.
       :param cls_meth: Indicates if the method is a class method."""

       def wrapper(instance_or_class, *args, **kwargs):
	   db = instance_or_class._get_db()
	   if isinstance(db, Database):
	       # the thing here is that a Sync method may be called by another
	       # Sync method and if that happens we simply execute method
	       r = method(instance_or_class, *args, **kwargs)

	   else:
	       # here we change the connection to a sync pymongo connection.
	       alias = utils.get_alias_for_db(db)
	       cls = instance_or_class if cls_meth else type(instance_or_class)
	       alias = utils.get_sync_alias(alias)
	       with switch_db(cls, alias):
		   r = method(instance_or_class, *args, **kwargs)

	   return r
       wrapper = functools.wraps(method)(wrapper)
       if cls_meth:
	   wrapper = classmethod(wrapper)
       return wrapper


   def _get_db(obj):
       """Returns the database connection instance for a given object."""

       if hasattr(obj, '_get_db'):
	   db = obj._get_db()

       elif hasattr(obj, 'document_type'):
	   db = obj.document_type._get_db()

       elif hasattr(obj, '_document'):
	   db = obj._document._get_db()

       elif hasattr(obj, 'owner_document'):
	   db = obj.owner_document._get_db()

       elif hasattr(obj, 'instance'):
	   db = obj.instance._get_db()

       else:
	   raise ConfusionError('Don\'t know how to get db for {}'.format(
	       str(obj)))

       return db


   def get_framework(obj):
       """Returns an asynchronous framework for a given object."""

       db = _get_db(obj)
       return db._framework


   def get_loop(obj):
       """Returns the io loop for a given object"""

       db = _get_db(obj)
       return db.get_io_loop()


   def get_future(obj, loop=None):
       """Returns a future for a given object"""

       framework = get_framework(obj)
       if loop is None:
	   loop = framework.get_event_loop()
       future = framework.get_future(loop)
       return future


   class switch_db(me_switch_db):

       def __init__(self, cls, db_alias):
	   """ Construct the switch_db context manager

	   :param cls: the class to change the registered db
	   :param db_alias: the name of the specific database to use
	   """
	   self.cls = cls
	   self.collection = cls._collection
	   self.db_alias = db_alias
	   self.ori_db_alias = cls._meta.get("db_alias", DEFAULT_CONNECTION_NAME)
	   self.patcher = MonkeyPatcher()

       def __enter__(self):
	   """ changes the db_alias, clears the cached collection and
	   patches _connections"""
	   super().__enter__()
	   self.patcher.patch_async_connections()
	   return self.cls

       def __exit__(self, t, value, traceback):
	   """ Reset the db_alias and collection """
	   self.cls._meta["db_alias"] = self.ori_db_alias
	   self.cls._collection = self.collection
	   self.patcher.__exit__(t, value, traceback)


   class OriginalDelegate(MotorAttributeFactory):

       """A descriptor that wraps a Motor method, such as insert or remove
       and returns the original PyMongo method. It still uses motor pool and
       event classes so it needs to run in a child greenlet.

       This is done  because I want to be able to asynchronize a method that
       connects to database but I want to do that in the mongoengine methods,
       the driver methods should work in a `sync` style, in order to not break
       the mongoengine code, but in a child greenlet to handle the I/O stuff.
       Usually is complementary to :class:`~mongomotor.metaprogramming.Async`.
       """

       def create_attribute(self, cls, attr_name):
	   return getattr(cls.__delegate_class__, attr_name)


   class Async(MotorAttributeFactory):

       """A descriptor that wraps a mongoengine method, such as save or delete
       and returns an asynchronous version of the method. Usually is
       complementary to :class:`~mongomotor.metaprogramming.OriginalDelegate`.
       """

       def __init__(self, cls_meth=False):
	   self.cls_meth = cls_meth
	   self.original_method = None

       def _get_super(self, cls, attr_name):
	   # Tries to get the real method from the super classes
	   method = None

	   for base in cls.__bases__:
	       try:
		   method = getattr(base, attr_name)
		   # here we use the __func__ stuff because we want to bind
		   # it to an instance or class when we call it
		   method = method.__func__
	       except AttributeError:
		   pass

	   if method is None:
	       raise AttributeError(
		   '{} has no attribute {}'.format(cls, attr_name))

	   return method

       def create_attribute(self, cls, attr_name):
	   self.original_method = self._get_super(cls, attr_name)
	   self.async_method = asynchronize(self.original_method,
					    cls_meth=self.cls_meth)
	   return self.async_method


   class Sync(Async):
       """A descriptor that wraps a mongoengine method, ensure_indexes
       and runs it using the synchronous pymongo driver.
       """

       def create_attribute(self, cls, attr_name):
	   method = self._get_super(cls, attr_name)
	   return synchronize(method, cls_meth=self.cls_meth)


   class AsyncWrapperMetaclass(type):
       """Metaclass for classes that use MotorAttributeFactory descriptors."""

       def __new__(cls, name, bases, attrs):

	   new_class = super().__new__(cls, name, bases, attrs)
	   for attr_name, attr in attrs.items():
	       if isinstance(attr, MotorAttributeFactory):
		   real_attr = attr.create_attribute(new_class, attr_name)
		   setattr(new_class, attr_name, real_attr)

	   return new_class


   class AsyncTopLevelDocumentMetaclass(AsyncWrapperMetaclass,
					TopLevelDocumentMetaclass):
       """Metaclass for top level documents that have asynchronous methods."""


   class AsyncGenericMetaclass(AsyncWrapperMetaclass):
       """Metaclass for any type of documents that use MotorAttributeFactory."""


   class AsyncDocumentMetaclass(AsyncWrapperMetaclass, DocumentMetaclass):
       """Metaclass for documents that use MotorAttributeFactory."""



.. code-block:: python

   # core.py

   import types
   from motor.core import (AgnosticCollection, AgnosticClient, AgnosticDatabase,
			   AgnosticCursor)
   from motor.metaprogramming import create_class_with_framework
   import pymongo
   from pymongo.database import Database
   from pymongo.collection import Collection
   from mongomotor.metaprogramming import OriginalDelegate


   def _rebound(ret, obj):
       try:
	   ret = types.MethodType(ret.__func__, obj)
       except AttributeError:
	   pass
       return ret


   class MongoMotorAgnosticCursor(AgnosticCursor):

       __motor_class_name__ = 'MongoMotorCursor'

       distinct = OriginalDelegate()
       explain = OriginalDelegate()

       def __init__(self, *args, **kwargs):
	   super(AgnosticCursor, self).__init__(*args, **kwargs)

	   # here we get the mangled stuff in the delegate class and
	   # set here
	   attrs = [a for a in dir(self.delegate) if a.startswith('_Cursor__')]
	   for attr in attrs:
	       setattr(self, attr, getattr(self.delegate, attr))

       # these are used internally. If you try to
       # iterate using for in a main greenlet you will
       # see an exception.
       # To iterate use a queryset and iterate using motor style
       # with fetch_next/next_object
       def __iter__(self):
	   return self

       def __next__(self):
	   return next(self.delegate)

       def __getitem__(self, index):
	   r = self.delegate[index]
	   if isinstance(r, type(self.delegate)):
	       # If the response is a cursor, transform it into a
	       # mongomotor cursor.
	       r = type(self)(r, self.collection)
	   return r

       # @aiter_compat
       # def __aiter__(self):
       #     return self

       # async def __anext__(self):
       #     # An optimization: skip the "await" if possible.
       #     if self._buffer_size() or await self.fetch_next:
       #         return self.next_object()
       #     raise StopAsyncIteration()


   class MongoMotorAgnosticCollection(AgnosticCollection):

       __motor_class_name__ = 'MongoMotorCollection'

       # Using the original delegate method (but with motor pool and event)
       # so I don't get a future as the return value and don't need to work
       # with mongoengine code.
       # insert = OriginalDelegate()
       insert_many = OriginalDelegate()
       insert_one = OriginalDelegate()
       # save = OriginalDelegate()
       # update = OriginalDelegate()
       update_one = OriginalDelegate()
       update_many = OriginalDelegate()
       find_one = OriginalDelegate()
       # find_and_modify = OriginalDelegate()
       find_one_and_update = OriginalDelegate()
       find_one_and_delete = OriginalDelegate()
       index_information = OriginalDelegate()

       def __init__(self, database, name, _delegate=None):

	   db_class = create_class_with_framework(
	       MongoMotorAgnosticDatabase, self._framework, self.__module__)

	   if not isinstance(database, db_class):
	       raise TypeError("First argument to MongoMotorCollection must be "
			       "MongoMotorDatabase, not %r" % database)

	   delegate = _delegate if _delegate is not None else\
	       Collection(database.delegate, name)
	   super(AgnosticCollection, self).__init__(delegate)
	   self.database = database

       def __getattr__(self, name):
	   if name.startswith('_'):
	       # Here first I try to get the _attribute from
	       # from the delegate obj.
	       try:
		   ret = getattr(self.delegate, name)
	       except AttributeError:
		   raise AttributeError(
		       "%s has no attribute %r. To access the %s"
		       " collection, use collection['%s']." % (
			   self.__class__.__name__, name, name,
			   name))
	       return _rebound(ret, self)

	   return self[name]

       def __getitem__(self, name):
	   collection_class = create_class_with_framework(
	       MongoMotorAgnosticCollection, self._framework, self.__module__)

	   return collection_class(self.database, self.name + '.' + name)

       def find(self, *args, **kwargs):
	   """Create a :class:`MongoMotorAgnosticCursor`. Same parameters as for
	   PyMongo's :meth:`~pymongo.collection.Collection.find`.

	   Note that ``find`` does not take a `callback` parameter, nor does
	   it return a Future, because ``find`` merely creates a
	   :class:`MongoMotorAgnosticCursor` without performing any operations
	   on the server.
	   ``MongoMotorAgnosticCursor`` methods such as
	   :meth:`~MongoMotorAgnosticCursor.to_list` or
	   :meth:`~MongoMotorAgnosticCursor.count` perform actual operations.
	   """
	   if 'callback' in kwargs:
	       raise pymongo.errors.InvalidOperation(
		   "Pass a callback to each, to_list, or count, not to find.")

	   cursor = self.delegate.find(*args, **kwargs)
	   cursor_class = create_class_with_framework(
	       MongoMotorAgnosticCursor, self._framework, self.__module__)

	   return cursor_class(cursor, self)


   class MongoMotorAgnosticDatabase(AgnosticDatabase):

       __motor_class_name__ = 'MongoMotorDatabase'

       dereference = OriginalDelegate()
       # authenticate = OriginalDelegate()

       def __init__(self, client, name, _delegate=None):
	   if not isinstance(client, AgnosticClient):
	       raise TypeError("First argument to MongoMotorDatabase must be "
			       "a Motor client, not %r" % client)

	   self._client = client
	   delegate = _delegate or Database(client.delegate, name)
	   super(AgnosticDatabase, self).__init__(delegate)

       def __getattr__(self, name):
	   if name.startswith('_'):
	       # samething. try get from delegate first
	       try:
		   ret = getattr(self.delegate, name)
	       except AttributeError:
		   raise AttributeError(
		       "%s has no attribute %r. To access the %s"
		       " collection, use database['%s']." % (
			   self.__class__.__name__, name, name,
			   name))
	       return _rebound(ret, self)

	   return self[name]

       def __getitem__(self, name):
	   collection_class = create_class_with_framework(
	       MongoMotorAgnosticCollection, self._framework, self.__module__)

	   return collection_class(self, name)

       def eval(self, code, *fields):
	   return self.command('eval', code, *fields)


   class MongoMotorAgnosticClientBase(AgnosticClient):

       # max_write_batch_size = ReadOnlyProperty()

       def __getattr__(self, name):
	   if name.startswith('_'):
	       # the same. Try get from delegate.
	       try:
		   ret = getattr(self.delegate, name)
	       except AttributeError:

		   raise AttributeError(
		       "%s has no attribute %r. To access the %s"
		       " database, use client['%s']." % (
			   self.__class__.__name__, name, name, name))

	       return _rebound(ret, self)

	   return self[name]

       def __getitem__(self, name):
	   db_class = create_class_with_framework(
	       MongoMotorAgnosticDatabase, self._framework, self.__module__)

	   return db_class(self, name)


   class MongoMotorAgnosticClient(MongoMotorAgnosticClientBase, AgnosticClient):

       __motor_class_name__ = 'MongoMotorClient'


Com esses dois caras aí - e mais uns *monkey patches* - o que eu fazia era estender
as classes do mongoengine, usava os *decorators* para deixar os métodos assíncronos
e assim eu não precisava escrever muito código, muito da lógica ainda ficava no
mongoengine. Algo assim:

.. code-block:: python

   class QuerySet(MEQuerySet, metaclass=AsyncGenericMetaclass):

       distinct = Async()
       explain = Async()
       in_bulk = Async()
       map_reduce = Async()
       modify = Async()
       update = Async()
       ...


Nem tudo ficava assim, alguns métodos eu tinha que meio que re-escrever algumas
coisas, mas já me poupava muita coisa.


Como ficou?
===========

Agora, sem o *pool* de *threads* e toda a metaprogramação pra deixar a coisa
assíncrona, eu precisei re-escrever as coisas de verdade. Agora, um método
que antes eu usava só um Async(), agora eu tive que implementar de
verdade, tipo assim:

.. code-block:: python

   ...
   async def distinct(self, field):
	   """Return a list of distinct values for a given field.

	   :param field: the field to select distinct values from

	   .. note:: This is a command and won't take ordering or limit into
	      account.
	   """
	   queryset = self.clone()

	   try:
	       field = self._fields_to_dbfields([field]).pop()
	   except LookUpError:
	       pass

	   raw_values = await queryset._cursor.distinct(field)
	   if not self._auto_dereference:
	       return raw_values

	   distinct = await self._dereference(
	       raw_values, 1, name=field, instance=self._document)

	   doc_field = self._document._fields.get(field.split(".", 1)[0])
	   instance = None

	   # We may need to cast to the correct type eg.
	   # ListField(EmbeddedDocumentField)
	   EmbeddedDocumentField = _import_class("EmbeddedDocumentField")
	   ListField = _import_class("ListField")
	   GenericEmbeddedDocumentField = _import_class(
	       "GenericEmbeddedDocumentField")
	   if isinstance(doc_field, ListField):
	       doc_field = getattr(doc_field, "field", doc_field)
	   if isinstance(doc_field, (EmbeddedDocumentField,
				     GenericEmbeddedDocumentField)):
	       instance = getattr(doc_field, "document_type", None)

	   # handle distinct on subdocuments
	   if "." in field:
	       for field_part in field.split(".")[1:]:
		   # if looping on embedded document, get the document
		   # type instance
		   if instance and isinstance(
		       doc_field, (EmbeddedDocumentField,
				   GenericEmbeddedDocumentField)
		   ):
		       doc_field = instance
		   # now get the subdocument
		   doc_field = getattr(doc_field, field_part, doc_field)
		   # We may need to cast to the correct type eg.
		   # ListField(EmbeddedDocumentField)
		   if isinstance(doc_field, ListField):
		       doc_field = getattr(doc_field, "field", doc_field)
		   if isinstance(
		       doc_field, (EmbeddedDocumentField,
				   GenericEmbeddedDocumentField)
		   ):
		       instance = getattr(doc_field, "document_type", None)

	   if instance and isinstance(
	       doc_field, (EmbeddedDocumentField, GenericEmbeddedDocumentField)
	   ):
	       distinct = [instance(**doc) for doc in distinct]

	   return distinct
	   ...

Eu basicamente peguei o método do mongoengine, copiei e alterei o que precisava
pra funcionar com o pymongo assíncrono. Muito do código do mongoengine que eu
só usava *de graça*, agora que eu copiei é minha responsabilidade manter, mas
isso tem até sua vantagem inesperada: Como o mongomotor é uma biblioteca que
já tem o que eu preciso, ela é mantida só atualizando as dependências e
corrigindo eventuais bugs. Isso, junto com o fato de que eu mexo nas entranhas
do mongoengine, acontece de dar uma quebradeira chata quando atualiza. Comigo
pegando mais código do mongoengine acredito que vai dar uma diminuída. Ainda
vai acontecer, porque ainda temos *monkey patches* e mexidas nas entranhas,
mas com o tempo e as correções que com certeza virão nas atualizações, tende
a estabilizar. Eu acho. Além do que o código assim fica muito mais simples
e fácil de achar os *bugs*.


Um pouco de história
====================

Eu falei que essa era a terceira encarnação do mongomotor. Já a primeira (por
volta de 2013) não era nem uma biblioteca separada, era um *package* dentro
de um projeto feito com tornado e a ideia era bem parecida com essa. Eu
simplesmente reimplementava o que precisava pra funcionar com o tornado.
Quando eventualmente quis usar o mesmo esquema em outro projeto - esse
com asyncio - foi que separei em uma biblioteca separada e fui ver como o motor
implementava o suporte a tornado e asyncio (que na época não tinha nem a
sintaxe de async/await). Quando vi como o motor funcionava pensei: "Porra, isso
é muito louco".

Foi assim que surgiu a segunda encarnação do mongomotor, com o metaprogramming
que mostrei no começo. Aquilo era tudo ideia que peguei do motor e adaptei pra
usar com o mongoengine. O pymongo tinha um parâmetro não documentado que era
basicamente feito pro motor usar. Se a memória não me falha, o parâmetro
chamava **_sockets** e tinha um comentário tipo: "não mexe se você não sabe
o que é isso". Basicamente tava lá só pro motor. O mongoengine não tinha nada
assim pra mim, então *monkey patch* foi mato.

Agora temos o terceiro mongomotor. Primeiro re-escrevi pra suportar asyncio,
agora pra tirar o motor. O que será depois? Só falta o mongoengine me
aprontar alguma.


Antes de ir embora
==================

Lembra que eu comentei que a api do motor e do pymongo async era teoricamente
a mesma? Então, não é! No motor o queryset.aggregate não é um método assíncrono
e no pymongo é. Tive que quebrar api do mongomotor.
