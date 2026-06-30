从零开始直到投入生产环境
====================================

我喜欢迅速行动。我想要我们的小项目尽快上线，比如现在就上线，投入生产环境中。我们现在还没有开发任何东西，那我们就从一个漂亮简约的“在建设中”页面开始。你会喜欢它的！

花点时间去试着找一张你满意的老式“在建设中” GIF 动图。我要用的是 `这张`_：

.. image:: images/under-construction.gif
    :align: center

我告诉过你了，这会非常有趣的。

初始化项目
---------------

我们之前已经一起安装了 ``symfony`` 命令行工具，现在用它来创建一个新的 Symfony 项目。

.. code-block:: terminal

    $ symfony new guestbook --version=8.1 --php=8.5 --webapp --docker --upsun
    $ cd guestbook

这个命令对 ``Composer`` 做了一层简单的封装，用来方便地创建 Symfony 项目。它使用一个包含了最少依赖的 `项目骨架`_，这些依赖是几乎所有项目都要用到的 Symfony 组件：一个控制台工具和创建 web 应用必需的 HTTP 抽象。

由于我们要创建一个功能完备的 web 应用，我们添加了几个能让我们更轻松的选项：

* ``--webapp``：默认情况下，会创建一个依赖尽可能少的应用。对大多数 web 项目来说，建议在此之上使用 ``webapp`` 包。它包含了“现代” web 应用所需的大部分包。``webapp`` 包会添加很多 Symfony 包，包括 Symfony Messenger 以及通过 Doctrine 使用的 PostgreSQL。

* ``--docker``：在你的本地电脑上，我们会用 Docker 来管理像 PostgreSQL 这样的服务。这个选项会启用 Docker，这样 Symfony 就会根据所需的包自动添加 Docker 服务（比如添加 ORM 时添加一个 PostgreSQL 服务，或者添加 Symfony Mailer 时添加一个邮件捕获器）。

* ``--upsun``：如果你想把项目部署到 Upsun，这个选项会自动生成一份合理的 Upsun 配置。Upsun 是在云端部署测试、预发布和生产环境 Symfony 应用的首选且最简单的方式。

如果你去看一下这个项目骨架在 GitHub 上的代码仓库，你会注意到它只有一个 ``composer.json`` 文件，几乎是空的。但是 ``guestbook`` 目录下有好多文件了，这是怎么做到的？答案就在 ``symfony/flex`` 这个包里。Symfony Flex 是一个 Composer 插件，它可以介入到包的安装过程中。当它检测到新装的包对应有一个 *recipe*，这个插件就会执行它。

Symfony Recipe 的主入口是一个 manifest 文件，它描述了在 Symfony 应用中自动注册一个包时所进行的操作。你在 Symfony 应用下安装一个包的时候，永远不需要去阅读包的 README 文件。自动化是 Symfony 的重要特点。

因为 Git 已经在你的电脑上安装过了，所以 ``symfony new`` 也会为我们创建一个 Git 仓库，并且会进行首次代码提交。

看一下目录结构：

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

``bin/`` 目录包含了主要的命令行入口：``console`` 程序。你会一直用到它。

``config/`` 目录由一套配置文件组成，它们提供了合理的缺省值。每个包对应一个配置文件。你很少会改动它们，信赖缺省值几乎总是个好主意。

``public/`` 目录是 web 根目录，下面的 ``index.php`` 脚本是所有 HTTP 动态资源的主入口。

``src/`` 目录存放你将要写的代码；你大部分时间都会花在里面。默认情况下，该目录下所有类都是用 ``App`` 作为命名空间。这里是你的主目录、你的代码、你的业务逻辑。Symfony 把这个目录下的主动权留给你。

``var/`` 目录包含了缓存，日志和其它应用程序运行时产生的文件。你可以不用管它。它是生产环境下唯一一个需要写权限的目录。

``vendor/`` 目录包含了所有由 Composer 安装的包，包括 Symfony 本身。这个目录是提高生产率的秘密武器。我们不要重新发明轮子，而是要依赖已有的程序库来完成那些困难的工作。这个目录由 Composer 管理，永远不要去动它。

这就是你目前所需知道的一切。

创建一些公开可访问资源
---------------------------------

``public/`` 目录下的任何文件都是可以通过浏览器直接访问的。比如说，如果你把你的 GIF 动图文件（把它命名为 ``under-construction.gif``）移动到新建的 ``public/images/`` 目录下，它就可以通过诸如 ``https://localhost/images/under-construction.gif`` 这样的 URL 来访问。

在这里下载我的 GIF 图：

.. code-block:: terminal

    $ mkdir public/images/
    $ php -r "copy('https://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

启动本地 web 服务器
--------------------------

.. index::
    single: Symfony CLI;server:start

``symfony`` 命令自带了一个为开发工作优化过的 web 服务器。如果我和你说它和 Symfony 搭配得很好，你不会惊讶的。但注意，不要在生产环境中使用这个 web 服务器。

在项目目录下，以后台运行的方式启动 web 服务器（用 ``-d`` 选项）：

.. code-block:: terminal

    $ symfony server:start -d

web 服务器启动了，它会使用从 8000 端口开始的第一个可用端口。你可以执行如下命令作为快捷方式，从而在浏览器里打开网站：

.. code-block:: terminal
    :class: ignore

    $ symfony open:local

你的默认浏览器应该会获得焦点，并且新开一个标签页，展示出和下图类似的内容：

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    运行 ``symfony server:log`` 来排查问题，它会把来自 web 服务器，PHP 和应用程序的最新日志显示出来。

浏览到 ``/images/under-construction.gif``。看上去像是这样吗？

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

满意了吧？让我们来提交我们的代码。

.. code-block:: terminal
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

为生产环境做准备
------------------------

.. index::
    single: Upsun;Initialization

把我们的工作发布到生产环境中，你觉得如何？我知道，我们甚至还没有一个正式的 HTML 页面来欢迎我们的用户。但哪怕是在生产服务器上看到那个”在建设中“的图片，也是向前迈进的伟大一步。你知道这句格言：*尽早部署，经常部署*。

你可以把你的应用放在任何支持 PHP 的主机供应商那里......这意味着市面上几乎所有的主机供应商。但还是要做几点检查：我们想要最新的 PHP 版本，而且需要有数据库、消息队列和其它更多的服务。

我做了我的选择，那就是用 `Upsun`_。它提供了我们所需的一切，而且它的收入也会用来资助 Symfony 的开发。

.. index::
    single: Symfony CLI;project:init

由于我们在创建项目时使用了 ``--upsun`` 选项，Upsun 已经初始化好了，它所需的唯一一个配置文件就是 ``.upsun/config.yaml``。

进入生产环境
------------------

.. index::
    single: Symfony CLI;cloud:project:create
    single: Symfony CLI;cloud:push

部署的时刻？

新建一个远程的 Upsun 项目：

.. code-block:: terminal

    $ symfony cloud:project:create --title="Guestbook" --plan=development

这个命令会做很多工作：

* 你第一次执行这个命令的时候，如果还没有用你的 Upsun 凭证来认证过身份的话，请做一下。

* 它会在 Upsun 上创建一个新项目（你创建的第一个项目可享有 30 天的 *免费* 使用期）。

然后就部署：

.. code-block:: terminal

    $ symfony cloud:push

把代码推送到 Git 代码仓库后，它就会部署好。在该命令执行后，这个项目就会有一个特定的域名，可以用来访问这个项目。

.. index::
    single: Symfony CLI;cloud:url

检查一下，看一切是否顺利运行：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

你应该会看到一个 404 错误页面，但浏览 ``/images/under-construction.gif`` 的话，你应该会看到我们的图片。

请注意你在 Upsun 上不会看到那个漂亮的 Symfony 默认页面。为什么呢？你一会就会知道 Symfony 支持若干环境变量，而 Upsun 把代码自动部署在了生产环境下。

.. index::
    single: Symfony CLI;cloud:project:delete

.. tip::

    如果你要在 Upsun 上删除项目，就用 ``cloud:project:delete`` 命令。

.. sidebar:: 深入学习

    * `Symfony 官方 recipe`_ 和 `社区贡献的 recipe`_ 的代码仓库，你可以把自己的 recipe 提交到那里；

    * `Symfony 本地 web 服务器`_；

    * `Upsun 文档`_。

.. _`这张`: https://clipartmag.com/images/website-under-construction-image-6.gif
.. _`项目骨架`: https://github.com/symfony/skeleton
.. _`Upsun`: https://upsun.com/symfony/?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
.. _`Symfony 官方 recipe`: https://github.com/symfony/recipes
.. _`社区贡献的 recipe`: https://github.com/symfony/recipes-contrib
.. _`Symfony 本地 web 服务器`: https://symfony.com/doc/current/setup/symfony_server.html
.. _`Upsun 文档`: https://developer.upsun.com/docs/get-started/stacks/symfony/index?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
