راه‌اندازی یک پایگاه‌داده
==================================================

.. index::
    single: Database

وبسایت کنفرانس Guestbook، در حال گرفتن بازخورد‌ها در طول کنفرانس است. ما نیاز داریم کامنت‌هایی که توسط شرکت‌کنندگان کنفرانس اظهار شده است را در یک انبار دائمی ذخیره‌سازی کنیم.

یک کامنت به بهترین وجه توسط یک داده‌ساختار ثابت بیان می‌شود: یک نویسنده، رایانامه‌ی نویسنده، متن بازخورد و یک تصویر اختیاری. نوعی از داده که می‌تواند به بهترین وجه در یک موتور پایگاه‌داده‌ی رابطه‌ای سنتی ذخیره شود.

PostgreSQL موتور پایگاه‌داده‌ای است که ما از آن استفاده خواهیم کرد.

افزودن PostgreSQL به Docker Compose
-------------------------------------------

.. index::
    single: Docker;PostgreSQL

ما تصمیم گرفتیم که در رایانه‌ی محلی‌مان، از Docker برای مدیریت سرویس‌ها استفاده کنیم. فایل ``compose.yaml`` تولیدشده از پیش PostgreSQL را به عنوان یک سرویس در بر دارد:

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

این یک سرور PostgreSQL را نصب خواهد کرد و همچنین تعدادی متغیر محیط را برای کنترل نام پایگاه‌داده و اعتبارنامه‌ها (credentials)، پیکربندی می‌کند. مقادیر آن واقعاً اهمیتی ندارند.

همچنین درگاه (port) مربوط به کانتینر PostgreSQL (یعنی درگاه ``5432``) را در اختیار میزبان محلی می‌گذاریم. این کمک می‌کند تا از طریق رایانه‌ی محلی به پایگاه‌داده دسترسی داشته باشیم:

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

    هنگامی که PHP در گام قبلی تنظیم شد، افزونه‌ی ``pdo_pgsql`` می‌بایست نصب می‌گردید.

اجرای Docker Compose
-------------------------

Docker Compose را در پس زمینه (``-d``) اجرا کنید:

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

مقداری صبر کنید و اجازه دهید تا پایگاه‌داده راه‌اندازی شود. سپس بررسی کنید که همه چیز به صورت صحیح در حال اجرا است:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

اگر هیچ کانتینری در حال اجرا نیست یا ستون ``State`` دارای مقدار ``Up`` نیست، لاگ‌های Docker را بررسی کنید:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

دسترسی به پایگاه‌داده‌ی محلی
-------------------------------------------------------

گاهی اوقات استفاده از ابزار خط فرمان ``psql``، می‌تواند مفید واقع شود. اما شما باید اعتبارنامه‌ها و نام پایگاه‌داده را به خاطر بیاورید. همچنین باید درگاه محلی پایگاه‌داده‌ی در حال اجرا بر روی میزبان را بدانید. Docker یک درگاه تصادفی را انتخاب می‌کند تا بتوانید به صورت همزمان بر روی بیش از یک پروژه که از PostgreSQL استفاده می‌کند، کار کنید (درگاه محلی بخشی از خروجی فرمان ``docker compose ps`` است).

اگر فرمان ``psql`` را از طریق رابط خط فرمان سیمفونی اجرا کنید، نیاز نیست که هیچ چیزی را به خاطر آورید.

رابط خط فرمان سیمفونی، به صورت خودکار تمام سرویس‌هایی که پروژه را اجرا می‌کنند، شناسایی کرده و متغیر‌های محیط لازم برای فرمان ``psql`` جهت اتصال به پایگاه‌داده را در اختیار فرمان قرار می‌دهد.

.. index::
    single: Symfony CLI;run psql

به کمک این قراردادها، دسترسی به پایگاه‌داده از طریق ``symfony run`` بسیار راحت‌تر است:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    اگر فایل اجرایی ``psql`` را بر روی میزبان محلی‌تان ندارید، می‌توانید آن را از طریق ``docker compose`` نیز اجرا کنید:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

دامپ و بازیابی داده‌های پایگاه‌داده
---------------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

برای دامپ‌کردن داده‌های پایگاه‌داده از ``pg_dump`` استفاده کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

و داده‌ها را بازیابی کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

افزودن PostgreSQL به Upsun
-----------------------------------------

.. index::
    single: Upsun;PostgreSQL

برای زیرساخت عمل‌آوری بر روی Upsun، افزودن یک سرویس همچون PostgreSQL، باید از طریق فایل ``.upsun/config.yaml`` انجام شود، که از پیش از طریق recipe بسته‌ی ``webapp`` انجام شده است:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

سرویس ``database`` یک پایگاه‌داده‌ی PostgreSQL است (همان نسخه‌ای که در Docker استفاده شد). Upsun در نخستین استقرار، دیسک آن را به صورت خودکار تخصیص می‌دهد؛ در صورت نیاز بعداً آن را با ``symfony cloud:resources:set`` تنظیم کنید.

همچنین نیاز داریم تا پایگاه‌داده را به کانتینر اپلیکیشن «متصل (link)» کنیم، که این موضوع در فایل ``.upsun/config.yaml`` توصیف شده است:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

سرویس ``database`` از نوع ``postgresql`` در کانتینر اپلیکیشن به عنوان ``database`` ارجاع داده شده است.

بررسی کنید که افزونه‌ی ``pdo_pgsql`` از پیش برای PHP نصب شده است:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

دسترسی به پایگاه‌داده‌ی Upsun
-----------------------------------------------------------

حالا PostgreSQL هم بر روی رایانه‌ی محلی‌تان از طریق Docker و هم در محیط عمل‌آوری بر روی Upsun در حال اجرا است.

همانطور که مشاهده کردیم، به لطف متغیر‌های محیط ارائه‌شده توسط ``symfony run``، اجرای ``symfony run psql`` به صورت خودکار به پایگاه‌داده‌ی میزبانی‌شده توسط Docker متصل می‌شود.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

اگر می‌خواهید به PostgreSQL میزبانی‌شده در کانتینرهای عمل‌آوری متصل شوید، می‌توانید یک تونل SSH میان رایانه‌ی محلی و زیرساخت Upsun برقرار کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

به صورت پیش‌فرض، سرویس‌های Upsun به صورت متغیر‌های محیط در اختیار رایانه‌ی محلی‌تان نیستند. اگر خلاف این را می‌خواهید، باید صریحاً فرمان ``var:expose-from-tunnel`` را اجرا کنید. چرا؟ اتصال به پایگاه‌داده‌ی عمل‌آوری، عملیاتی خطرناک است. شما ممکن است با داده‌های *واقعی* خرابکاری کنید.

حالا از طریق فرمان ``symfony run psql``، همچون گذشته به پایگاه‌داده‌ی PostgreSQL ریموت متصل شوید:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

فراموش نکنید که پس از پایان کار، تونل را ببندید:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    برای اجرای تعدادی پرس‌وجوی SQL بر روی پایگاه‌داده‌ی عمل‌آوری، به جای استفاده از شِل (shell)، می‌توانید از فرمان ``symfony sql`` نیز استفاده کنید.

ارائه‌ی متغیر‌های محیط
--------------------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

به لطف متغیر‌های محیط، Docker Compose و Upsun به صورت یکپارچه با یکدیگر کار می‌کنند.

با اجرای فرمان ``symfony var:export`` تمام متغیر‌های محیط ارائه‌شده توسط ``symfony`` را بررسی کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

متغیر‌های محیط ``PG*`` توسط ابزار ``psql`` خوانده می‌شوند. سایر موارد چطور؟

زمانی که یک تونل به Upsun همراه با ``var:expose-from-tunnel`` باز است، فرمان ``var:export`` متغیر‌های محیط ریموت را بازمی‌گرداند:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

توصیف زیرساخت شما
------------------------------

شاید هنوز متوجه آن نشده باشید، اما ذخیره‌ی زیرساخت در فایل‌هایی در کنار کد، کمک زیادی می‌کند. Docker و Upsun از فایل‌های پیکربندی برای توصیف زیرساخت پروژه استفاده می‌کنند. هنگامی که یک ویژگی جدید به یک سرویس اضافه نیاز دارد، تغییرات کد و تغییرات زیرساخت بخشی از یک patch واحد هستند.

.. sidebar:: بیشتر بدانید

    * `سرویس‌های Upsun`_؛

    * `تونل Upsun`_؛

    * `مستندات PostgreSQL`_؛

    * `فرمان‌های Docker Compose`_.

.. _`سرویس‌های Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`تونل Upsun`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`مستندات PostgreSQL`: https://www.postgresql.org/docs/
.. _`فرمان‌های Docker Compose`: https://docs.docker.com/reference/cli/docker/compose/
