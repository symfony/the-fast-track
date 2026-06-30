استفاده از Redis برای ذخیره‌ی نشست‌ها
=========================================================

.. index::
    single: Redis

بسته به ترافیک وب‌سایت و/یا زیرساخت آن، ممکن است بخواهید به جای PostgreSQL از Redis برای مدیریت نشست‌های کاربران استفاده کنید.

زمانی که درباره‌ی انشعاب کد پروژه برای انتقال نشست‌ها از فایل‌سیستم به پایگاه‌داده صحبت کردیم، تمام گام‌های لازم برای افزودن یک سرویس جدید را فهرست کردیم.

در اینجا نحوه‌ی افزودن Redis به پروژه‌تان در یک patch آمده است:

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

*زیبا* نیست؟

برای راه‌اندازی سرویس Redis، Docker را «بازراه‌اندازی» کنید:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

با مرور وب‌سایت، آن را به صورت محلی بیازمایید؛ همه چیز باید مانند گذشته کار کند.

طبق معمول commit و مستقر کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push

.. sidebar:: بیشتر بدانید

    * `مستندات Redis`_.

.. _`مستندات Redis`: https://redis.io/documentation
