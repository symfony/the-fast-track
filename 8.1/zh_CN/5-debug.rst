排查问题
============

拥有正确的工具对问题进行调试，这也是设置项目的一部分。幸运的是，很多好用的辅助工具已经作为 ``webapp`` 包的一部分包含进来了。

学习 Symfony 的调试工具
------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

作为开始，当你需要去找到程序的问题根源时，Symfony 分析器（profiler）可以为你节省大量时间。

看一下首页，你会看到窗口底部有一个工具栏：

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

可能你第一个注意到的就是红色的 **404**。记住这个页面是一个占位页，因为我们还没有开发首页。尽管这个缺省的欢迎页很漂亮，它仍然是一个错误页面，所以当前的 HTTP 状态码是 404，而不是 200。多亏了这个 *web 调试工具栏*，你一下子就得到了这些信息。

如果你点击那个小的感叹号，你会看到“真正”的异常信息，这些信息也是 Symfony 分析器日志的一部分。如果你要查看堆栈跟踪，点击左侧菜单的“Exception”链接。

无论什么时候你的代码出了问题，你总是能看到类似下图的异常页面，它提供了你所需的一切信息，来让你明白这个问题以及它的源头：

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

你可以在这个工具栏里到处点击，花点时间探索一下 Symfony 分析器提供的各种信息。

.. index::
    single: Symfony CLI;server:log

日志对于调试也很有用。Symfony 有一个很方便的命令可以查看最新的日志（来自 web 服务器、PHP 和应用程序的日志）：

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

我们来做个小实验。打开 ``public/index.php``，破坏掉里面的PHP代码（比如在代码中间加个 foobar）。在浏览器中刷新页面，然后观察日志输出流：

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

输出的内容被很美观地高亮了，这样你就可以注意到错误信息。

理解 Symfony 环境
---------------------

.. index::
    single: Symfony Environments

因为 Symfony 分析器只是在开发时有用，所以我们要避免把它装在生产环境中。默认情况下，Symfony 只会为 ``dev`` 和 ``test`` 环境自动安装它。

Symfony 支持 *环境* 这个概念。默认情况下，它内置支持 3 种环境：``dev``、``prod`` 和 ``test``，但你也能添加任意多的环境。所有的环境共享同一套代码，但它们代表了不同的 *配置*。

举个例子，在 ``dev`` 环境中，所有的调试工具会启用，而在 ``prod`` 环境中，应用的性能会被优化。

改变 ``APP_ENV`` 这个环境变量，就可以从一个环境切换到另一个。

当你部署到 Upsun 上时，环境会被自动切换到 ``prod`` （这个值存储在 ``APP_ENV`` 里）。

管理环境配置
------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` 可以在终端里用“真正”的环境变量来设置：

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

在生产服务器上，用真正的环境变量来设置类似 ``APP_ENV`` 这样的值是推荐的做法。但在开发机器上，定义很多环境变量会有点繁琐，所以我们在一个 ``.env`` 文件中定义它们。

当新建项目时，一个带有合理初始值的 ``.env`` 文件会自动生成：

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    借助 Symfony Flex 使用的 recipe，任何包都可以添加新的环境变量到这个文件中。

``.env`` 文件会被提交到代码仓库，它的内容是生产环境下的 *默认* 值。你可以通过新建 ``.env.local`` 文件来覆盖那些默认值。这个 ``.env.local`` 文件不应该被提交到 Git，这也是为什么 ``.gitignore`` 文件已经将它忽略了。

绝不要把机密或敏感的数据存储在这些文件中。我们会在后面看到如何管理机密内容。

配置你的 IDE
----------------

开发环境下有异常抛出时，Symfony 会显示一个带有异常信息和堆栈跟踪的页面。当显示文件路径时，它会自动增加一个链接，这个链接可以在你首选的 IDE 中打开该文件，并定位到正确的行。为了使用这个功能，你需要去配置你的 IDE。Symfony 自带支持很多 IDE；我在这个项目里使用的是 Visual Studio Code：

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

链接的文件不仅限于异常。例如在配置了 IDE 后，*web 调试工具栏* 里的控制器也会变成可点击的。

在生产环境中排错
------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:logs
    single: Symfony CLI;cloud:ssh

在生产服务器上调试总是更为棘手。比如，你无法使用 Symfony 分析器。日志提供的信息也更少。但是显示最近的日志是可以的：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --tail

你甚至可以通过 SSH 连接到 web 容器：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

别担心，你不太会把东西弄坏。大部分文件都是只读的。在生产环境中你无法进行热修复，但你在本书后面会学到一个好得多的方法。

.. sidebar:: 深入学习

    * `SymfonyCasts：环境和配置文件教程`_；

    * `SymfonyCasts ：环境变量教程`_；

    * `SymfonyCasts 的 web 调试工具栏和分析器教程`_；

    * 在 Symfony 应用中 `管理多个 .env 文件`_。

.. _`SymfonyCasts：环境和配置文件教程`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`SymfonyCasts ：环境变量教程`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`SymfonyCasts 的 web 调试工具栏和分析器教程`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`管理多个 .env 文件`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
