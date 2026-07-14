استخدام Redis لتخزين الجلسات
================================================

.. index::
    single: Redis

اعتمادًا على حركة مرور موقع الويب و / أو بنيته التحتية ، قد ترغب في استخدام Redis لإدارة جلسات المستخدم بدلاً من PostgreSQL.

عندما تحدثنا عن تفريع كود المشروع لنقل الجلسة من نظام الملفات إلى قاعدة البيانات ، قمنا بإدراج جميع الخطوات المطلوبة لإضافة خدمة جديدة.

إليك كيفية إضافة Redis إلى مشروعك في حزمة واحدة:

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

أليست *جميلة* ؟

"إعادة تشغيل" Docker لبدء خدمة Redis:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

اختبر محليًا من خلال تصفح الموقع ؛ يجب أن يعمل كل شيء كما كان من قبل.

التحزيم والنشر كالمعتاد:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: الذهاب أبعد من ذلك

    * `Redis docs`_.

.. _`Redis docs`: https://redis.io/documentation
