检查你的开发环境
========================

在开始投入到这个项目之前，我们需要检查下，让你有一个好的开发环境。这很重要。和 10 年前相比，现在可供使用的开发者工具已经非常不同了。它们朝好的方向演进了很多。如果不使用它们，那就太可惜了。好的工具可以帮你走得很远。

请不要跳过这一步，或者至少读一下关于 ``symfony`` 命令的最后一段。

一台电脑
------------

你需要一台电脑。好消息是你可以使用任何一个流行的操作系统：macOS、Windows 或者 Linux。Symfony 和我们要使用的所有工具都兼容这些操作系统。

基于我个人观点的选择
------------------------------

我想要挑选最好的工具来快速推进项目。所以我基于自己的判断为本书选择了如下的配套开发软件。

我们选择用 `PostgreSQL`_ 来作为数据库、消息队列、缓存和会话存储。对大多数项目来说，PostgreSQL 是最好的解决方案，它的伸缩性很棒，而且由于只要管理一个服务，它简化了基础设施的搭建。

在书的结尾，我们会学习如何用 `RabbitMQ`_ 来处理消息队列，以及如何用 `Redis`_ 来存储会话。

IDE
---

.. index:: IDE

如果你想的话，你可以使用 Notepad 作为编辑器，但我并不推荐。

我过去用 Textmate 开发，现在不用了。一个“真正”的 IDE 所带来的开发舒适感是无价的。自动补齐、自动添加并排序 ``use`` 语句、从一个文件跳转到另一个，这些都是可以大幅提高生产率的功能。

我会推荐 `Visual Studio Code`_ 和 `PhpStorm`_。前者是免费的，后者不免费，但和Symfony整合得更好（多亏了 `Symfony Support Plugin`_ 插件）。你来选择好了。我知道你想要问我用的是哪个 IDE。我是在 Visual Studio Code 里写作本书的。

终端
------

.. index:: Terminal

我们会一直从 IDE 切换到命令行。你可以使用你 IDE 里内置的终端，但我更想用一个单独的终端，这样它的窗口空间也更大。

Linux 自带 ``Terminal`` 。在 macOS 上可以用 `iTerm2`_ 。Windows 上 `Hyper`_ 也很好用。

Git
---

.. index:: Git

在版本控制方面，我们将使用 `Git`_，因为现在所有人都在用它。

在 Windows 上，安装 `Git bash`_。

确保你了解那些常见操作，比如运行 ``git clone``、``git log``、``git show``、``git diff``、``git checkout``......

PHP
---

.. index::
    single: PHP
    single: PHP extensions

我们会用 Docker 来提供服务，但考虑到性能、稳定性和简洁性，我还是喜欢在本地电脑安装 PHP。你愿意的话可以叫我守旧派，但对我来说，本地 PHP 和 Docker 服务的组合堪称完美。

请使用 PHP 8.5 版本，并且确保以下 PHP 扩展已经安装，如果没有的话现在就把它们装好：``intl``、``pdo_pgsql``、``xsl``、``amqp``、``gd``、``openssl``、``sodium`` 和 ``iconv``。可选择安装 ``redis``、 ``curl`` 和 ``zip``。

你可以用 ``php -m`` 命令来查看当前启用的扩展。

如果你的平台支持 ``php-fpm`` 的话，我们也需要它，用 ``php-cgi`` 也可以.

Composer
--------

.. index:: Composer

如今依赖包管理是 Symfony 项目的关键。`Composer`_ 是 PHP 的包管理器，请去获取它的最新版本。

如果你对 Composer 还不熟悉的话，去花点时间读一下关于它的内容。

.. tip::

    你不必输入完整的命令名：``composer req`` 和 ``composer require`` 的作用是一样的，你也可以用 ``composer rem`` 代替 ``composer remove``......

Docker 和 Docker Compose
-------------------------

.. index:: Docker,Docker Compose

服务将由 Docker 和 Docker Compose 来管理。`把它们安装好`_  并且启动 Docker。如果你是第一次使用 Docker，去熟悉一下这个工具。不用害怕，我们对它的使用十分直截了当。没有花哨的配置项，也没有复杂的设置步骤。

Symfony CLI
-----------

.. index:: Symfony CLI

最后但同样重要的一点是，我们会使用 ``symfony`` 命令来提高生产率。它提供了本地 web 服务器，可以与 Docker 完整集成，并通过 Upsun 提供云端支持，这些都可以为我们节省很多时间。

现在就安装 `Symfony CLI`_。

要在本地使用 HTTPS，我们也需要去 `安装一个 certificate authority (CA)`_ 来启动对 TLS 的支持。运行下面的命令：

.. index::
    single: Symfony CLI;server:ca:install

.. code-block:: terminal
    :class: ignore

    $ symfony server:ca:install

运行下面的命令，来检查你的电脑是否符合所需的全部条件：

.. code-block:: terminal
    :class: ignore

    $ symfony book:check-requirements

如果你想让开发更有趣，你也可以运行 `Symfony proxy`_。这个步骤是可选的，但它可以让你在项目中使用一个以 ``.wip`` 结尾的本地域名。

当我们在终端执行命令的时候，我们几乎总是会加上 ``symfony`` 前缀，比如我们会用 ``symfony composer``，而不仅仅是 ``composer``，又比如我们用 ``symfony console`` 来代替 ``./bin/console``。

这样做的主要原因是 ``symfony`` 命令会自动设置一些环境变量，它们是根据你电脑上 Docker 运行的服务来设置的。本地 web 服务器会把这些环境变量自动注入到 HTTP 请求中。所以，在命令行上使用 ``symfony`` 可以确保在各处都有一致的行为。

此外，``symfony`` 命令会为项目在一切可能的 PHP 版本中，自动选择“最好”的版本。

.. _`PostgreSQL`: https://www.postgresql.org/
.. _`RabbitMQ`: https://www.rabbitmq.com/
.. _`Redis`: https://redis.io/
.. _`Visual Studio Code`: https://code.visualstudio.com/
.. _`PhpStorm`: https://www.jetbrains.com/phpstorm/
.. _`Composer`: https://getcomposer.org/
.. _`把它们安装好`: https://docs.docker.com/install/
.. _`Symfony CLI`: https://symfony.com/download
.. _`安装一个 certificate authority (CA)`: https://symfony.com/doc/current/setup/symfony_server.html#enabling-tls
.. _`Symfony Support Plugin`: https://plugins.jetbrains.com/plugin/7219-symfony-support
.. _`iTerm2`: https://iterm2.com/
.. _`Hyper`: https://hyper.is/
.. _`Git`: https://git-scm.com/
.. _`Git bash`: https://gitforwindows.org/
.. _`Symfony proxy`: https://symfony.com/doc/current/setup/symfony_server.html#setting-up-the-local-proxy
