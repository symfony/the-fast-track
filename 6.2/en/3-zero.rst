Going from Zero to Production
=============================

I like to go fast. I want our little project to be live as fast as possible. Like now. In production. As we haven't developed anything yet, we will start by deploying a nice and simple "Under construction" page. You will love it!

Spend some time trying to find the ideal, old fashioned, and animated "Under construction" GIF on the Internet. Here is `the one`_ I'm going to use:

.. image:: images/under-construction.gif
    :align: center

I told you, it is going to be a lot of fun.

Initializing the Project
------------------------

Create a new Symfony project with the ``symfony`` CLI tool we have previously installed together:

.. code-block:: terminal

    $ symfony new guestbook --version=6.2 --php=8.1 --webapp --docker --cloud
    $ cd guestbook

This command is a thin wrapper on top of ``Composer`` that eases the creation of Symfony projects. It uses a `project skeleton`_ that includes the bare minimum dependencies; the Symfony components that are needed for almost any project: a console tool and the HTTP abstraction needed to create Web applications.

As we are creating a fully-featured web application, we have added a few options that will make our life easier:

* ``--webapp``: By default, an application with the fewest possible dependencies is created. For most web projects, it is recommended to use the ``webapp`` package on top. It contains most of the packages needed for "modern" web applications. The ``webapp`` package adds a lot of Symfony packages, including Symfony Messenger and PostgreSQL via Doctrine.

* ``--docker``: On your local machine, we will use Docker to manage services like PostgreSQL. This option enables Docker so that Symfony will automatically add Docker services based on the required packages (a PostgreSQL service when adding the ORM or a mail catcher when adding Symfony Mailer for instance).

* ``--cloud``: If you want to deploy your project on Platform.sh, this option automatically generates a sensible Platform.sh configuration. Platform.sh is the preferred and simplest way to deploy testing, staging, and production Symfony environments in the cloud.

If you have a look at the GitHub repository for the skeleton, you will notice that it is almost empty. Just a ``composer.json`` file. But the ``guestbook`` directory is full of files. How is that even possible? The answer lies in the ``symfony/flex`` package. Symfony Flex is a Composer plugin that hooks into the installation process. When it detects a package for which it has a *recipe*, it executes it.

The main entry point of a Symfony Recipe is a manifest file that describes the operations that need to be done to automatically register the package in a Symfony application. You never have to read a README file to install a package with Symfony. Automation is a key feature of Symfony.

As Git is installed on our machine, ``symfony new`` also created a Git repository for us and it added the very first commit.

Have a look at the directory structure:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

The ``bin/`` directory contains the main CLI entry point: ``console``. You will use it all the time.

The ``config/`` directory is made of a set of default and sensible configuration files. One file per package. You will barely change them, trusting the defaults is almost always a good idea.

The ``public/`` directory is the web root directory, and the ``index.php`` script is the main entry point for all dynamic HTTP resources.

The ``src/`` directory hosts all the code you will write; that's where you will spend most of your time. By default, all classes under this directory use the ``App`` PHP namespace. It is your home. Your code. Your domain logic. Symfony has very little to say there.

The ``var/`` directory contains caches, logs, and files generated at runtime by the application. You can leave it alone. It is the only directory that needs to be writable in production.

The ``vendor/`` directory contains all packages installed by Composer, including Symfony itself. That's our secret weapon to be more productive. Let's not reinvent the wheel. You will rely on existing libraries to do the hard work. The directory is managed by Composer. Never touch it.

That's all you need to know for now.

Creating some Public Resources
------------------------------

Anything under ``public/`` is accessible via a browser. For instance, if you move your animated GIF file (name it ``under-construction.gif``) into a new ``public/images/`` directory, it will be available at a URL like ``https://localhost/images/under-construction.gif``.

Download my GIF image here:

.. code-block:: terminal

    $ mkdir public/images/
    $ php -r "copy('http://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

Launching a Local Web Server
----------------------------

.. index::
    single: Symfony CLI;server:start

The ``symfony`` CLI comes with a Web Server that is optimized for development work. You won't be surprised if I tell you that it works nicely with Symfony. Never use it in production though.

From the project directory, start the web server in the background (``-d`` flag):

.. code-block:: terminal

    $ symfony server:start -d

The server started on the first available port, starting with 8000. As a shortcut, open the website in a browser from the CLI:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local

Your favorite browser should take the focus and open a new tab that displays something similar to the following:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    To troubleshoot problems, run ``symfony server:log``; it tails the logs from the web server, PHP, and your application.

Browse to ``/images/under-construction.gif``. Does it look like this?

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

Satisfied? Let's commit our work:

.. code-block:: terminal
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

Preparing for Production
------------------------

.. index::
    single: Platform.sh;Initialization

What about deploying our work to production? I know, we don't even have a proper HTML page yet to welcome our users. But being able to see the little "under construction" image on a production server would be a great step forward. And you know the motto: *deploy early and often*.

You can host this application on any provider supporting PHP... which means almost all hosting providers out there. Check a few things though: we want the latest PHP version and the possibility to host services like a database, a queue, and some more.

I have made my choice, it's going to be `Platform.sh`_. It provides everything we need and it helps fund the development of Symfony.

.. index::
    single: Symfony CLI;project:init

As we used the ``--cloud`` option when we created the project, Platform.sh has already been initialized with a few files needed by Platform.sh, namely ``.platform/services.yaml``, ``.platform/routes.yaml``, and ``.platform.app.yaml``.

Going to Production
-------------------

.. index::
    single: Symfony CLI;cloud:project:create
    single: Symfony CLI;cloud:deploy

Deploy time?

Create a new remote Platform.sh project:

.. code-block:: terminal

    $ symfony cloud:project:create --title="Guestbook" --plan=development

This command does a lot:

* The first time you launch this command, authenticate with your Platform.sh credentials if not done already.

* It provisions a new project on Platform.sh (you get 30 days *for free* on the first project you create).

Then, deploy:

.. code-block:: terminal

    $ symfony cloud:deploy

The code is deployed by pushing the Git repository. At the end of the command, the project will have a specific domain name you can use to access it.

.. index::
    single: Symfony CLI;cloud:url

Check that everything worked fine:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

You should get a 404, but browsing to ``/images/under-construction.gif`` should reveal our work.

Note that you don't get the beautiful default Symfony page on Platform.sh. Why? You will learn soon that Symfony supports several environments and Platform.sh automatically deployed the code in the production environment.

.. index::
    single: Symfony CLI;cloud:project:delete

.. tip::

    If you want to delete the project on Platform.sh, use the ``cloud:project:delete`` command.

.. sidebar:: Going Further

    * The repositories for the `official Symfony recipes`_ and for the `recipes contributed by the community`_, where you can submit your own recipes;

    * The `Symfony Local Web Server`_;

    * The `Platform.sh documentation`_.

.. _`the one`: http://clipartmag.com/images/website-under-construction-image-6.gif
.. _`project skeleton`: https://github.com/symfony/skeleton
.. _`Platform.sh`:     https://platform.sh/marketplace/symfony/?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
.. _`official Symfony recipes`: https://github.com/symfony/recipes
.. _`recipes contributed by the community`: https://github.com/symfony/recipes-contrib
.. _`Symfony Local Web Server`: https://symfony.com/doc/current/setup/symfony_server.html
.. _`Platform.sh documentation`: https://docs.platform.sh/guides/symfony.html?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
