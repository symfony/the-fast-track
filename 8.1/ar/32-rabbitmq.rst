استخدام RabbitMQ كوسيط الرسائل
=================================================

.. index::
    single: RabbitMQ

RabbitMQ هو وسيط رسائل شائع جدًا يمكنك استخدامه كبديل لـ PostgreSQL.

التحول من PostgreSQL إلى RabbitMQ
--------------------------------------------

لاستخدام RabbitMQ بدلاً من PostgreSQL كوسيط للرسائل:

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

نحتاج أيضًا إلى إضافة دعم RabbitMQ إلى Messenger:

.. code-block:: terminal

    $ symfony composer req amqp-messenger

إضافة RabbitMQ إلى Docker Stack
---------------------------------------

.. index::
    single: Docker;RabbitMQ

كما قد تكون خمنت ، نحتاج أيضًا إلى إضافة RabbitMQ إلى مكدس Docker Compose:

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

إعادة تشغيل خدمات Docker
---------------------------------------

لإجبار Docker Compose على أخذ حاوية RabbitMQ في الاعتبار ، أوقف الحاويات وأعد تشغيلها:

.. code-block:: terminal

    $ docker compose stop
    $ docker compose up -d --remove-orphans

.. code-block:: terminal
    :class: hide

    $ sleep 10

استكشاف واجهة إدارة الويب RabbitMQ
--------------------------------------------------------

.. index::
    single: Symfony CLI;open:local:rabbitmq

إذا كنت تريد رؤية قوائم الانتظار والرسائل التي تتدفق عبر RabbitMQ ، فافتح واجهة إدارة الويب الخاصة به:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:rabbitmq

أو من شريط أدوات تصحيح أخطاء الويب:

.. figure:: screenshots/rabbitmq-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

استخدم ``guest``/``guest`` لتسجيل الدخول إلى واجهة مستخدم إدارة RabbitMQ:

.. figure:: screenshots/rabbitmq-management.png
    :alt: /
    :align: center
    :figclass: with-browser

نشر RabbitMQ
---------------

.. index::
    single: Upsun;RabbitMQ
    single: RabbitMQ

يمكن إضافة RabbitMQ إلى خوادم الإنتاج عن طريق إضافته إلى قائمة الخدمات:

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

قم بالرجوع إليها في تكوين حاوية الويب أيضًا وقم بتمكين ``amqp`` امتداد PHP :

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

عندما يتم تثبيت خدمة RabbitMQ على مشروع ، يمكنك الوصول إلى واجهة إدارة الويب الخاصة به عن طريق فتح النفق أولاً:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony open:remote:rabbitmq

    # when done
    $ symfony cloud:tunnel:close

.. sidebar:: الذهاب أبعد من ذلك

    * `RabbitMQ docs`_.

.. _`RabbitMQ docs`: https://www.rabbitmq.com/documentation.html
