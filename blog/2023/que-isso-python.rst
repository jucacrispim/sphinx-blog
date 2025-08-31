Que isso, Python?
=================

.. post:: Apr 23, 2023
   :category: computisses
   :author: Juca Crispim


Essa é foda. O Python me mostrando traceback errado agora. Como pode?

.. code-block:: sh

    Task exception was never retrieved
    future: <Task finished name='Task-6939' coro=<BuildExecuter._run_build() done, defined at /home/pdjexception=KeyError('8ba5a94b-93bc-4a27-a4e4-05dc82c785cd')>
    Traceback (most recent call last):
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/build.py, line 848, in _run_build
	await slave.build(build, **self.repository.envvars)
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/slave.py, line 367, in build
	build, envvars=envvars):
	     ^^^^^^^^^^^^^^^^^^^
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/client.py, line 125, in build
	validate_cert=validate_cert)
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/slave.py, line 409, in _process_info
	async def _process_build_info(self, build, repo, build_info):
		    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/slave.py, line 575, in _process_step_output_info
	async def _get_step(self, build, step_uuid, wait=False):
	    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File /home/pdj/.virtualenvs/toxicbuild/lib/python3.11/site-packages/toxicbuild/master/slave.py, line 560, in _update_build_step_info
	return True
	    ^^^^^^^^
    KeyError: '8ba5a94b-93bc-4a27-a4e4-05dc82c785cd'

Na verdade esse aí foi o seguinte: Atualizei um código e antes de eu reuniciar
o servidor deu um erro, então o python tá mostrando o erro em um fonte
diferente. Mas foi engraçado. :P
