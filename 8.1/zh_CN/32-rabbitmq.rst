使用 RabbitMQ 作为消息代理
==================================

.. index::
    single: RabbitMQ

RabbitMQ 是一个非常流行的消息代理，你可以用它来代替 PostgreSQL。

从 PostgreSQL 切换到 RabbitMQ
---------------------------------

要使用 RabbitMQ 作为消息代理来代替 PostgreSQL：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -5,7 +5,7 @@ framework:
             transports:
                 # https://symfony.com/doc/current/messenger.html#transport-configuration
                 async:
    -                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
    +                dsn: '%env(RABBITMQ_URL)%'
                     retry_strategy:
                         max_retries: 3
                         multiplier: 2

我们还需要为 Messenger 添加 RabbitMQ 支持：

.. code-block:: terminal

    $ symfony composer req amqp-messenger

把 RabbitMQ 加进 Docker 栈：
---------------------------------

.. index::
    single: Docker;RabbitMQ

你可能已经猜到了，我们仍然需要把 RabbitMQ 加入到 Docker Compose 栈：

.. code-block:: diff
    :caption: patch_file

    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -18,6 +18,10 @@ services:
         image: redis:8.0-alpine
         ports: [6379]

    +  rabbitmq:
    +    image: rabbitmq:4.2-management
    +    ports: [5672, 15672]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:

重启 Docker 服务：
-----------------------

为了强制让 RabbitMQ 容器在 Docker Compose 中生效，把容器都关闭，并且重启它们：

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

.. code-block:: terminal
    :class: hide

    $ sleep 10

探索 RabbitMQ 的网站管理界面
-------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

如果你想要看到 RabbitMQ 中往来的队列和消息，打开它的网站管理界面：

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

或者从网站调试工具栏打开：

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

使用 ``guest``/``guest`` 作为账号密码来登录 RabbitMQ 的管理界面：

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

部署 RabbitMQ
---------------

.. index::
    single: Upsun;RabbitMQ
    single: RabbitMQ

将 RabbitMQ 加入服务列表，就可以把它加进生产服务器。

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -25,4 +25,8 @@ services:
             rediscache:
                 type: redis:8.0

    +    queue:
    +        type: rabbitmq:4.2
    +        size: S
    +
     applications:

也可以在网站容器配置中找到它，然后启动 ``amqp`` 这个 PHP 扩展：

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -39,6 +39,7 @@ applications:

             runtime:
                 extensions:
    +                - amqp
                     - apcu
                     - blackfire
                     - ctype
    @@ -72,5 +73,6 @@ applications:
             relationships:
                 database: "database:postgresql"
                 redis: "rediscache:redis"
    +            rabbitmq: "queue:rabbitmq"

             hooks:
                 build: |

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;open:remote:rabbitmq

当项目中安装了 RabbitMQ 服务，你可以通过先打开隧道来访问它的网站管理界面：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: 深入学习

    * `RabbitMQ 文档`_。

.. _`RabbitMQ 文档`: https://www.rabbitmq.com/documentation.html
