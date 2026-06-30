设置数据库
===============

.. index::
    single: Database

这个会议留言本网站要收集关于会议的反馈。我们要把参会人员贡献的评论保存在一个持久存储中。

如下这个固定的数据结构可以很好地描述一条评论：留言者、电子邮箱、反馈的文字和一张可选的照片。这种类型的数据最适合存放在传统的关系型数据库中。

PostgreSQL 是我们选用的数据库。

把 PostgreSQL 添加到 Docker Compose
---------------------------------------

.. index::
    single: Docker;PostgreSQL

在我们的本地电脑上，我们决定用 Docker 来管理服务。生成的 ``compose.yaml`` 文件里已经包含了 PostgreSQL 作为一个服务：

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

这个步骤会安装一个 PostgreSQL 服务器，并配置一些环境变量来设置数据库名和账号密码。具体的值不重要。

我们把容器的 PostgreSQL 端口（``5432``）暴露给本地机器，这样它就可以连接到数据库了：

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    在前面安装 PHP 的步骤里，``pdo_pgsql`` 扩展应该已经装好了。

启动 Docker Compose
---------------------

在后台运行 Docker Compose（使用 ``-d`` 选项）：

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

先稍等一下，让数据库启动，然后检查是否一切正常：

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

如果没有容器在运行，或者 ``State`` 栏不是显示为 ``Up``，检查一下 Docker Compose 的日志：

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

连入本地数据库
---------------------

你时不时会用到 ``psql`` 这个命令行小工具。但是你需要记住数据库名以及账号密码。更加不太为人所熟悉的一点是，你还需要知道数据库运行时的本地端口。Docker 会选用一个随机端口，这样你就可以同时开发多个使用了 PostgreSQL 的项目（本地端口是 ``docker compose ps`` 命令输出中的一部分）。

如果你在 ``symfony`` 命令里使用 ``psql``，你就不必记住任何信息。

``symfony`` 命令会自动检测到项目运行的 Docker 服务，它会把 ``psql`` 连接数据库所需的环境变量暴露出来。

.. index::
    single: Symfony CLI;run psql

多亏这些约定，使用 ``symfony run`` 来连接数据库方便多了：

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    如果你在本机上没有 ``psql`` 这个二进制文件，你也可以通过 ``docker compose`` 来运行它：

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

导出和恢复数据库的数据
---------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

使用 ``pg_dump`` 来导出数据库的数据：

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

并且恢复数据：

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

在 Upsun 中加入 PostgreSQL
-------------------------------------

.. index::
    single: Upsun;PostgreSQL

在 Upsun 的生产环境软件设施里，添加一个诸如 PostgreSQL 这样的服务，是通过修改 ``.upsun/config.yaml`` 文件来完成的，而这一步已经由 ``webapp`` 包的 recipe 帮我们做好了：

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

这个 ``database`` 服务是一个 PostgreSQL 数据库（和 Docker 里的版本一样）。Upsun 会在第一次部署时自动为它分配磁盘空间；如有需要，之后可以用 ``symfony cloud:resources:set`` 来调整。

我们也需要把数据库“链接”到应用的容器，这同样描述在 ``.upsun/config.yaml`` 文件里：

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

在应用容器中，``postgresql`` 类型的 ``database`` 服务被引用为 ``database``。

检查一下 ``pdo_pgsql`` 扩展是否已经为 PHP 运行时安装好了：

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

连入 Upsun 数据库
-----------------------------

现在 PostgreSQL 在本地借助 Docker 运行，同时也在 Upsun 的生产环境中运行。

正如我们所见，运行 ``symfony run psql`` 会自动连接到 Docker 里的数据库，这要归功于 ``symfony run`` 暴露出的环境变量。

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

如果你想要连接到生产环境容器中的 PostgreSQL，你需要打开一个 SSH 隧道来接通你的本地电脑和 Upsun 平台设施：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

默认情况下，Upsun 服务不会在你的本地电脑上以环境变量暴露出来。要想把它们暴露出来，你必须明确运行 ``var:expose-from-tunnel`` 命令。为什么要这样呢？连接到生产数据库是一个危险的操作。你可能会破坏 *真实的* 数据。

现在，用 ``symfony run psql`` 连接到远程的 PostgreSQL 服务器，和以前一样：

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

连接完后，不要忘了关掉隧道：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    如果要在生产数据库上直接运行一些 SQL 语句，而不是获得一个 shell，你可以用 ``symfony sql`` 命令。

暴露环境变量
------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

借助于环境变量，Docker Compose 和 Upsun 可以和 Symfony 无缝对接。

执行 ``symfony var:export`` 可以查看 ``symfony`` 暴露出的所有环境变量：

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

``psql`` 小工具会读取 ``PG*`` 形式的环境变量。那其它环境变量呢？

如果一条隧道通过 ``var:expose-from-tunnel`` 连接到了 Upsun，那么 ``var:export`` 命令就会返回远程的环境变量：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

描述你的基础设施
------------------------

你可能还没意识到，把基础设施信息存储在文件中，并和代码放在一起，会有很多好处。Docker 和 Upsun 用配置文件来描述项目的基础设施。当一个新功能需要一个额外服务时，代码的改动和基础设施信息的改动会来自同一个补丁。

.. sidebar:: 深入学习

    * `Upsun 服务`_；

    * `Upsun 隧道`_；

    * `PostgreSQL 文档`_；

    * `Docker Compose 命令`_。

.. _`Upsun 服务`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Upsun 隧道`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL 文档`: https://www.postgresql.org/docs/
.. _`Docker Compose 命令`: https://docs.docker.com/reference/cli/docker/compose/
