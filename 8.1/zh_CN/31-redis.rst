使用 Redis 存储会话
=========================

.. index::
    single: Redis

根据网站的流量以及它的基础设施，你可能想用 Redis 取代 PostgreSQL 来管理用户的会话。

之前我们谈到新建一个项目的代码分支来把会话从文件系统迁移到数据库时，我们列出了添加一个新服务的所有必要步骤。

只在一个代码补丁中把 Redis 加入到你项目里，可以这样做：

.. index::
    single: Session;Redis
    single: Redis
    single: Docker;Redis
    single: Upsun;Redis

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -37,6 +37,7 @@ applications:
                     - iconv
                     - mbstring
                     - pdo_pgsql
    +                - redis
                     - sodium
                     - xsl

    @@ -62,6 +63,7 @@ applications:

             relationships:
                 database: "database:postgresql"
    +            redis: "rediscache:redis"

             hooks:
                 build: |
    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -21,3 +21,6 @@ services:
                 type: network-storage:2.0

    +    rediscache:
    +        type: redis:8.0
    +
     applications:
    --- i/compose.yaml
    +++ w/compose.yaml
    @@ -14,6 +14,10 @@ services:
           # - ./docker/db/data:/var/lib/postgresql/data:rw
     ###< doctrine/doctrine-bundle ###

    +  redis:
    +    image: redis:8.0-alpine
    +    ports: [6379]
    +
     volumes:
     ###> doctrine/doctrine-bundle ###
       database_data:
    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -4,3 +4,3 @@ framework:
         # Note that the session will be started ONLY if you read or write from it.
         session:
    -        handler_id: '%env(resolve:DATABASE_URL)%'
    +        handler_id: '%env(REDIS_URL)%'

这难道不 *漂亮* 吗？

“重启” Docker 来启动 Redis 服务：

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

通过浏览网站来在本地测试；一切应该如之前一样正常运行。

提交并且像往常一样部署：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: 深入学习

    * `Redis 文档`_。

.. _`Redis 文档`: https://redis.io/documentation
