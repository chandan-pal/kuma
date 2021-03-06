===========
Development
===========

Pick an Environment
===================
There are two development environments, and you need to install at
least one of them first.

* The :doc:`Vagrant-managed VM <installation>` is the mature development
  environment, but we are experimenting with other options. It can be tricky to
  provision. The "all-in-one" VM style can sometimes be a poor model for the
  multi-machine production environment. During the transition, some
  documentation will assume it is the only environment.
* The :doc:`Docker containerized environment <installation-docker>` is the new
  development environment. The Docker images are already provisioned, so setup
  is faster. It is a better model for the production environment, and after the
  planned rehost will be almost exactly the same. It does not yet have all of
  the features of the Vagrant environment, and it is currently slower for many
  development tasks.

Vagrant and Docker can be used at the same time on the same "host" machine (your
laptop or desktop computer).

Basic Vagrant Usage
-------------------
Edit files as usual on your host machine; the current directory is
mounted via NFS at ``/home/vagrant/src`` within the VM. Updates should be
reflected without any action on your part. Useful vagrant sub-commands::

    vagrant ssh     # Connect to the VM via ssh
    vagrant suspend # Sleep the VM, saving state
    vagrant halt    # Shutdown the VM
    vagrant up      # Boot up the VM
    vagrant destroy # Destroy the VM

Run all commands in this doc on the VM after ``vagrant ssh``.

Basic Docker Usage
------------------
Edit files as usual on your host machine; the current directory is mounted
via Docker host mounting at ``/app`` within the ``kuma_web_1`` and
other containers. Useful docker sub-commands::

    docker exec -it kuma_web_1 bash  # Start an interactive shell
    docker logs kuma_web_1           # View logs from the web container
    docker-compose logs -f           # Continuously view logs from all containers
    docker restart kuma_web_1        # Force a container to reload
    docker-compose stop              # Shutdown the containers
    docker-compose up -d             # Start the containers
    docker-compose rm                # Destroy the containers

There are ``make`` shortcuts on the host for frequent commands, such as::

    make up         # docker-compose up -d
    make bash       # docker exec -it kuma_web_1 bash
    make shell_plus # docker exec -it kuma_web_1 ./manage.py shell_plus

Run all commands in this doc in the ``kuma_web_1`` container after ``make bash``

Running Kuma
============
The Vagrant environment runs everything in a single VM. It runs MySQL,
ElasticSearch, Apache, and other "backend" services whenever the VM is running.
There are additional Kuma-specific services that are configured in
``Procfile``, and are run with::

    foreman start

The development instance is then available at https://developer-local.allizom.org.

When the Docker container environment is started (``make up`` or similar), all
of the services are also started. The development instance is available at
http://localhost:8000.

Running the Tests
=================
A great way to check that everything really is working is to run the test
suite.

Django tests
------------
Run the Django test suite::

    make test

For more information, see the :doc:`test documentation <tests>`.

Front-end tests
---------------
To run the front-end (selenium) tests, see :doc:`Client-side Testing with
Intern <tests-ui>`.

Kumascript tests
----------------
If you're changing Kumascript, be sure to run its tests too.
See https://github.com/mozilla/kumascript

Compiling Stylus Files
======================
Stylus files need to be compiled for changes to take effect.

In the Vagrant environment, the ``foreman`` task ``stylus`` will automatically
compile Stylus files when they change, placing the generated CSS files at
``build/assets/css``.

In either environment, compilation can be run manually::

    scripts/compile-stylesheets

To watch for changes to the files and recompile::

    scripts/compile-stylesheets -w

Watching for file changes performs well in the Vagrant environment, but can be
slow with the host-mounted files in the Docker container.

Database Migrations
===================
Apps are migrated using Django's migration system. To run the migrations::

    manage.py migrate

If your changes include schema modifications, see the Django documentation for
the `migration workflow`_.

.. _migration workflow: https://docs.djangoproject.com/en/1.8/topics/migrations/#workflow

Coding Conventions
==================
See CONTRIBUTING.md_ for details of the coding style on Kuma.

If you're expecting ``reverse`` to return locales in the URL
(``/en-US/docs/Mozilla`` versus ``/docs/Mozilla``), use ``LocalizingClient``
instead of the default client for the ``TestCase`` class.

.. _CONTRIBUTING.md: https://github.com/mozilla/kuma/blob/master/CONTRIBUTING.md

Managing Dependencies
=====================

Python dependencies
-------------------
Kuma tracks its Python dependencies with pip_.  See the
`README in the requirements folder`_ for details.

.. _pip: https://pip.pypa.io/
.. _README in the requirements folder: https://github.com/mozilla/kuma/tree/master/requirements

Front-end dependencies
----------------------
Front-end dependencies are managed by Bower_ and checked into the repository.
Follow these steps to add or upgrade a dependency:

#. On the host, update ``bower.json``
#. (*Docker only*) In the container, install ``git`` (``apt-get install -y git``)
#. (*Docker only*) In the container, install ``bower-installer`` (``npm install -g bower-installer``)
#. In the VM or container, install the dependency (``bower-installer``)
#. On the host, prepare the dependency to be committed (``git add path/to/dependency``)

Front-end dependencies that are not already managed by Bower should begin using
this approach the next time they're upgraded.

.. _Bower: http://bower.io

Advanced Configuration
======================
`Environment variables`_ are used to change the way different components works.
There are a few ways to change an environment variables:

* Exporting in the shell, such as::

    export DEBUG=True;
    ./manage.py runserver

* A one-time override, such as::

    DEBUG=True ./manage.py runserver

* Changing the ``environment`` list in ``docker-compose.yml``.
* Creating a ``.env`` file in the repository root directory.

.. _Environment variables: http://12factor.net/config

.. _vagrant-config:

The Vagrant Environment
-----------------------
It is easiest to configure Vagrant with a ``.env`` file, so that overrides are used
when ``vagrant up`` is called.  A sample ``.env`` could contain::

    VAGRANT_MEMORY_SIZE=4096
    VAGRANT_CPU_CORES=4
    # Comments are OK, for documentation and to disable settings
    # VAGRANT_ANSIBLE_VERBOSE=true

Configuration variables that are available for Vagrant:

- ``VAGRANT_NFS``

  Default: ``true`` (Windows: ``false``)
  Whether or not to use NFS for the synced folder.

- ``VAGRANT_MEMORY_SIZE``

  The size of the Virtualbox VM memory in MB. Default: ``2048``

- ``VAGRANT_CPU_CORES``

  The number of virtual CPU core the Virtualbox VM should have. Default: ``2``

- ``VAGRANT_IP``

  The static IP the Virtualbox VM should be assigned to. Default: ``192.168.10.55``

- ``VAGRANT_GUI``

  Whether the Virtualbox VM should boot with a GUI. Default: ``false``

- ``VAGRANT_ANSIBLE_VERBOSE``

  Whether the Ansible provisioner should print verbose output. Default: ``false``

- ``VAGRANT_CACHIER``

  Whether to use the ``vagrant-cachier`` plugin to cache system packages
  between installs. Default: ``true``

.. _advanced_config_docker:

The Docker Environment
----------------------
Running docker-compose_ will create and run several containers, and each
container's environment and settings are configured in ``docker-compose.yml``.
The settings are "baked" into the containers created by ``docker-compose up``,

To override a container's settings for development, use a local override file.
For example, the ``web`` service runs in container ``kuma_web_1`` with the
default command 
"``gunicorn -w 4 --bind 0.0.0.0:8000 --timeout=120 kuma.wsgi:application``".
A useful alternative for debugging is to run a single-threaded process that
loads the Werkzeug debugger on exceptions (see docs for runserver_plus_), and
that allows for stepping through the code with a debugger.
To use this alternative, create an override file ``docker-compose.dev.yml``::

    version: "2"
    services:
      web:
        command: ./manage.py runserver_plus 0.0.0.0:8000
        stdin_open: true
        tty: true


This is similar to "``docker run -it <image> ./manage.py runserver_plus``",
using all the other configuration items in ``docker-compose.yml``.
Apply the custom setting with::

    docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

You can then add ``pdb`` breakpoints to the code
(``import pdb; pdb.set_trace``) and connect to the debugger with::

    docker attach kuma_web_1

To always include the override compose file, add it to your ``.env`` file::

    COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml

A similar method can be used to override environment variables in containers,
run additional services, or make other changes.  See the docker-compose_
documentation for more ideas on customizing the Docker environment.

.. _docker-compose: https://docs.docker.com/compose/overview/
.. _pdb: https://docs.python.org/2/library/pdb.html
.. _runserver_plus: http://django-extensions.readthedocs.io/en/latest/runserver_plus.html

The Database
------------
The database connection is defined by the environment variable
``DATABASE_URL``, with these defaults::

    DATABASE_URL=mysql://kuma:kuma@localhost:3306/kuma              # Vagrant
    DATABASE_URL=mysql://root:kuma@mysql:3306/developer_mozilla_org # Docker

The format is defined by the dj-database-url_ project::

    DATABASE_URL=mysql://user:password@host:port/database

If you configure a new database, override ``DATABASE_URL`` to connect to it. To
add an empty schema to a freshly created database::

    ./manage.py migrate

To connect to the database specified in ``DATABASE_URL``, use::

    ./manage.py dbshell

.. _dj-database-url: https://github.com/kennethreitz/dj-database-url

Asset Generation
----------------
Kuma will automatically run in debug mode, with the ``DEBUG`` setting
turned to ``True``. That will make it serve images and have the pages
formatted with CSS automatically.

Setting ``DEBUG=false`` file will put the installation in production mode and
ask for minified assets.  This only works in the Vagrant environment, which
uses Apache to serve the static files.  In Docker, static files will not be
served and the site will be unstyled.

Production assets
*****************
Assets are compressed on production. To emulate production and test compressed
assets locally (*Vagrant only*):

#. Set the environment variables ``DEBUG=false``
#. Run ``make compilejsi18n collectstatic`` in the VM or container
#. Restart the web process by retarting ``foreman``

Secure Cookies
--------------
To prevent error messages like "``Forbidden (CSRF cookie not set.):``", set the
environment variable::

    CSRF_COOKIE_SECURE = false

This is the default in Docker, which does not support local development with
HTTPS.
