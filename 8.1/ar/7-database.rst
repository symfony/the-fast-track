إعداد قاعدة بيانات
==================================

.. index::
    single: Database

يدور موقع سجل زوار المؤتمر حول جمع التعليقات أثناء المؤتمرات. نحتاج إلى تخزين التعليقات التي ساهم بها حضور المؤتمر في مخزن دائم.

يتم وصف التعليق بشكل أفضل من خلال بنية بيانات ثابتة: المؤلف ، بريده الإلكتروني ، نص الملاحظات ، وصورة اختيارية. نوع البيانات التي يمكن تخزينها بشكل أفضل في محرك قاعدة بيانات علائقية تقليدي.

PostgreSQL هو محرك قاعدة البيانات الذي سنستعمل.

إضافة PostgreSQL  إلى Docker Compose
--------------------------------------------

.. index::
    single: Docker;PostgreSQL

على الجهاز المحلي لدينا ، قررنا استخدام Docker لإدارة الخدمات. يحتوي ملف ``compose.yaml`` المُولّد بالفعل على PostgreSQL كخدمة:

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

سيؤدي هذا إلى تثبيت خادم PostgreSQL وتكوين بعض متغيرات البيئة التي تتحكم في اسم قاعدة البيانات وبيانات الاعتماد. القيم لا تهم حقا.

نكشف أيضًا منفذ PostgreSQL (`` 5432``) من الحاوية container إلى المضيف المحلي. سيساعدنا ذلك في الوصول إلى قاعدة البيانات من الجهاز الخاص بنا:

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

    يجب أن يكون قد تم تثبيت ملحق ``pdo_pgsql`` عندما تم إعداد PHP في خطوة سابقة.

بدء Docker Compose
---------------------

بدء Docker Compose في الخلفية (`` -d``):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

انتظر قليلاً للسماح بتشغيل قاعدة البيانات والتحقق من أن كل شيء يسير على ما يرام:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

إذا لم تكن هناك حاويات قيد التشغيل أو إذا كان عمود ``State`` لا يقرأ ``Up`` ، فتحقق من سجلات Docker Compose:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

الوصول إلى قاعدة البيانات المحلية
--------------------------------------------------------------

قد يكون استخدام الأداة المساعدة لسطر الأوامر ``psql`` مفيدًا من وقت لآخر. ولكن عليك أن تتذكر بيانات الاعتماد واسم قاعدة البيانات. أقل وضوحا ، تحتاج أيضا إلى معرفة المنفذ المحلي الذي تديره قاعدة البيانات على المضيف. يختار Docker منفذًا عشوائيًا حتى تتمكن من العمل على أكثر من مشروع باستخدام  PostgreSQL في نفس الوقت (المنفذ المحلي هو جزء من مخرجات ``docker compose ps``).

إذا قمت بتشغيل ``psql`` عبر Symfony CLI ، فلن تحتاج إلى تذكر أي شيء.

يقوم Symfony CLI بالكشف تلقائيًا عن خدمات Docker التي يتم تشغيلها للمشروع ويكشف متغيرات البيئة التي يحتاجها ``psql`` للاتصال بقاعدة البيانات.

.. index::
    single: Symfony CLI;run psql

بفضل هذه الاتفاقيات، أصبح الوصول إلى قاعدة البيانات عبر ``symfony run`` أسهل كثيرًا:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    إذا لم يكن لديك مشغل ``psql`` الثنائي على مضيفك المحلي ، يمكنك أيضًا تشغيله عبر ``docker compose``:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

فحص واستعادة بيانات قاعدة البيانات
----------------------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

استخدم ``pg_dump`` لتفريغ بيانات قاعدة البيانات:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

واستعادة البيانات:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

إضافة PostgreSQL إلى Upsun
-----------------------------------------

.. index::
    single: Upsun;PostgreSQL

بالنسبة للبنية التحتية للإنتاج على Upsun ، يجب إضافة خدمة مثل PostgreSQL في ملف ``.upsun/config.yaml`` ، وهو ما تم بالفعل عبر وصفة حزمة ``webapp``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

خدمة ``database`` هي قاعدة بيانات PostgreSQL (نفس النسخة المستخدمة في Docker). يخصص Upsun قرصها تلقائيًا عند أول نشر؛ عدّله لاحقًا باستخدام ``symfony cloud:resources:set`` إذا لزم الأمر.

نحتاج أيضًا إلى "ربط" قاعدة البيانات بحاوية التطبيق الموضحة في ``.upsun/config.yaml``:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

تتم الإشارة إلى خدمة ``database`` من النوع ``postgresql`` على أنها ``database`` في حاوية التطبيق.

تحقق من أن ملحق ``pdo_pgsql`` مثبّت بالفعل لوقت تشغيل PHP:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

الوصول إلى قاعدة بيانات Upsun
--------------------------------------------------------

يعمل PostgreSQL الآن محليًا عبر Docker وفي الإنتاج على Upsun.

كما رأينا للتو ، فإن تشغيل ``symfony run psql`` يتصل تلقائيًا بقاعدة البيانات التي يستضيفها Docker بفضل متغيرات البيئة المتاحة بواسطة ``symfony run``.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

إذا كنت ترغب في الاتصال بـ PostgreSQL المستضافة على حاويات الإنتاج ، يمكنك فتح نفق SSH بين الجهاز المحلي والبنية الأساسية لــ Upsun:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

بشكل افتراضي ، لا يتم كشف خدمات Upsun كمتغيرات بيئة على الجهاز المحلي. يجب عليك القيام بذلك بشكل صريح بتشغيل أمر ``var:expose-from-tunnel``. لماذا ؟ الاتصال بقاعدة بيانات الإنتاج عملية خطيرة. يمكنك العبث مع بيانات *حقيقية*.

الآن ، قم بالاتصال بقاعدة بيانات PostgreSQL عن بعد من خلال ``symfony run psql`` كما كان من قبل:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

عند الانتهاء ، لا تنس إغلاق النفق:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    لتشغيل بعض استعلامات SQL في قاعدة بيانات الإنتاج بدلاً من الحصول على محث الأوامر ، يمكنك أيضًا استخدام أمر``symfony sql``.

فضح متغيرات البيئة
----------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

يعمل Docker Compose و Upsun بسلاسة مع Symfony بفضل متغيرات البيئة.

تحقق من كل متغيرات البيئة المكشوفة بواسطة ``symfony`` عن طريق تنفيذ ``symfony var:export``:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

تتم قراءة متغيرات البيئة ``PG*`` بواسطة الأداة المساعدة ``psql``. ماذا عن الآخرين؟

عندما يكون النفق مفتوحًا لـ Upsun مع ``var:expose-from-tunnel``، فإن الأمر ``var:export`` يُرجع متغيرات البيئة البعيدة:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

وصف البنية التحتية الخاصة بك
----------------------------------------------------

ربما لم تكن قد أدركت ذلك حتى الآن ، لكن تخزين البنية التحتية في ملفات بجانب الكود يساعد كثيرًا. Docker و Upsun يستخدمان ملفات التكوين لوصف البنية التحتية للمشروع. عندما تحتاج ميزة جديدة إلى خدمة إضافية ، فإن التغييرات البرمجية وتغييرات البنية التحتية هي جزء من نفس التصحيح.

.. sidebar:: الذهاب أبعد من ذلك

    * `خدمات Upsun`_؛

    * `نفق Upsun`_؛

    * `PostgreSQL وثائق دعم`_؛

    * `أوامر Docker Compose`_.

.. _`خدمات Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`نفق Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL وثائق دعم`: https://www.postgresql.org/docs/
.. _`أوامر Docker Compose`: https://docs.docker.com/reference/cli/docker/compose/
