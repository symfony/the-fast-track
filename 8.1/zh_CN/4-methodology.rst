采用一套方法
==================

传授一门技能就是反反复复重复一件事情。我向你承诺，我不会这么做。在每一个步骤结尾，你需要动动手指保存你的工作。这就像用保存快捷键 ``Ctrl+S``，但这个保存针对网站的。

执行一个 Git 策略
-----------------------

.. index::
    single: Git;add
    single: Git;commit

在每个步骤的结尾，不要忘了提交你的改动：

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

你可以安全地把“所有文件”加入暂存区，因为 Symfony 为你管理了一个 ``.gitignore`` 文件。每个包都能增加配置项。来看一下当前的内容：

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

那些奇怪的字符串是 Symfony Flex 加上的标记，这样当你决定卸载一个依赖包时，Symfony Flex 就知道要移除哪些内容。所有繁琐的工作都是由 Symfony 完成的，而不是你。

把你的代码仓库推送到远程的服务器，这会是个好主意。GitHub、GitLab 或者 Bitbucket 都是不错的选择。

.. note::

    如果你部署到 Upsun，你已经有一份 Git 仓库的副本，因为当你使用 ``cloud:push`` 时，Upsun 在背后使用了 Git。但你不应该依赖 Upsun 的 Git 仓库。它只是用来部署，并不是一个备份。

持续地部署到生产环境
------------------------------

.. index::
    single: Symfony CLI;cloud:push

另外一个好习惯就是频繁地部署。在每个步骤末尾做次部署是一个不错的节奏。

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
