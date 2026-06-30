مدیریت کارایی
=========================

.. index::
    single: Blackfire
    single: Profiler

بهینه‌سازی قبل از موعد، ریشه‌ی تمام بدبختی‌ها است.

شاید شما قبلاً این نقل قول را خوانده باشید. اما دوست دارم آن را به طور کامل ذکر کنم:

ما باید بهبود‌های جزئی را فراموش کنیم، می‌توان در ۹۷ درصد موارد گفت: بهینه‌سازی قبل از موعد، ریشه‌ی تمام بدبختی‌ها است. البته نباید از فرصت‌های موجود در آن ۳ درصد حیاتی صرف نظر کنیم.

--   Donald Knuth

هر بهبود کارایی جزئی، به خصوص برای وب‌سایت‌های تجارت الکترونیک (e-commerce) می‌تواند تفاوت ایجاد کند. حالا اپلیکیشن guestbook برای نسختین بار آماده است، بیایید ببینیم چگونه می‌توان کارایی‌ آن را بررسی کرد.

بهترین راه برای پیدا کردن بهینه سازی‌های کارایی، استفاده از یک *نمایه‌ساز (profiler)* است. امروزه محبوب‌ترین آن‌ها `Blackfire`_ است (سلب مسئولیت تام: من بنیان‌گذار پروژه‌ی Blackfire نیز هستم).

معرفی Blackfire
--------------------

Blackfire از بخش‌های متعددی ساخته شده است:

* یک *client* که نمایه‌سازی را راه می‌اندازد (ابزار CLI مربوط به Blackfire یا یک افزونه‌ی مرورگر برای گوگل کروم یا فایرفاکس)؛

* یک *مأمور (agent)* که داده‌ها را برای ارسال به blackfire.io جهت نمایش، مهیا و تجمیع می‌کند؛

* یک افزونه‌ی PHP (*probe*) که کد PHP را به وسایل اندازه‌گیری مجهز می‌کند؛

برای کار با Blackfire، ابتدا لازم است که `ثبت‌نام کنید`_.

با اجرای اسکریپت نصب سریع زیر، Blackfire را بر روی رایانه‌ی محلی‌تان نصب کنید:

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

این نصاب، ابزار CLI مربوط به Blackfire را بارگیری و نصب می‌کند.

پس از پایان، PHP probe را بر روی تمام نسخه‌های PHP نصب کنید:

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

و PHP probe را برای پروژه‌مان فعال کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

وب سرور را بازراه‌اندازی کنید تا PHP بتواند Blackfire را بار بگیرد:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

ابزار CLI مربوط به Blackfire، نیاز دارد تا با اعتبارنامه‌های **client** شخصی‌تان پیکربندی شود (برای ذخیره‌ی نمایه‌های پروژه‌ی شما در اکانت شخصی‌تان). آن‌ها را در بالای `صفحه‌ی`_ ``Settings/Credentials`` پیدا کنید و فرمان‌های زیر را با جایگذاری مقادیر مربوطه، اجرا کنید:

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

تنظیم مأمور Blackfire بر روی Docker
--------------------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

سرویس مأمور Blackfire از پیش در پشته‌ی Docker Compose پیکربندی شده است:

.. code-block:: yaml
    :caption: compose.override.yaml
    :class: ignore

    ###> blackfireio/blackfire-symfony-meta ###
    blackfire:
        image: blackfire/blackfire:2
        # uncomment to store Blackfire credentials in a local .env.local file
        #env_file: .env.local
        environment:
        BLACKFIRE_LOG_LEVEL: 4
        ports: [8307]
    ###< blackfireio/blackfire-symfony-meta ###

برای ارتباط با سرور، شما نیاز دارید تا  اعتبارنامه‌های **server** شخصی‌تان را بدست آورید (این اعتبارنامه‌ها، جایی که می‌خواهید نمایه‌ها را در آن ذخیره کنید، مشخص می‌کنند -- می‌توانید برای هر پروژه یکی ایجاد کنید)؛ آن‌ها در پایین `صفحه‌ی`_ ``Settings/Credentials`` قابل یافتن هستند. آن‌ها را به عنوان secret ذخیره کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

حالا می‌توانید کانتینر جدید را راه‌اندازی کنید:

.. code-block:: terminal
    :class: ignore

    $ docker compose stop
    $ docker compose up -d --remove-orphans

تعمیر  یک نصبِ خرابِ Blackfire
----------------------------------------------

اگر هنگام profiling خطا دریافت می‌کنید، سطح لاگ را در Blackfire افزایش دهید تا اطلاعات بیشتری در لاگ‌ها دریافت کنید:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/php.ini
    +++ w/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

وب سرور را بازراه‌اندازی کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

و لاگ‌ها را دنبال کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

مجدداً نمایه‌سازی کرده و خروجی لاگ را بررسی کنید.


پیکربندی Blackfire در محیط عمل‌آوری
----------------------------------------------------------

.. index::
    single: Upsun;Blackfire

Blackfire به صورت پیش‌فرض در تمام پروژه‌های Upsun قرار داده شده است.

اعتبارنامه‌های *server* را به عنوان secretهای **عمل‌آوری** تنظیم کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

PHP probe از پیش مانند سایر افزونه‌های PHP موردنیاز فعال شده است:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 9
    :class: ignore

    runtime:
        extensions:
            - apcu
            - blackfire
            - ctype
            - iconv
            - mbstring
            - pdo_pgsql
            - sodium
            - xsl

پیکربندی Varnish برای Blackfire
-------------------------------------------

.. index::
    single: Upsun;Varnish

قبل از اینکه بتوانید مستقر کنید و profiling را شروع کنید، نیاز دارید که از نهان‌سازی HTTP توسط Varnish عبور کنید (آن را دور بزنید). اگر اینکار را نکنید، Blackfire هرگز نمی‌تواند به اپلیکیشن PHP دست یابد. شما تنها درخواست‌های نمایه‌سازی‌ای که از رایانه‌ی محلی‌تان می‌آید را مجاز خواهید شمرد.

IP فعلی‌تان را پیدا کنید:

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

و از آن برای پیکربندی Varnish استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "192.168.0.1";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

حالا می‌توانید استقرار را انجام دهید.

نمایه‌سازی صفحات وب
-------------------------------------

.. index::
    single: Profiling;Web Pages

شما می‌توانید صفحات وب سنتی را از طریق فایرفاکس یا گوگل کروم و از طریق `افزونه‌ی اختصاصی آن‌ها`_ ، نمایه‌سازی کنید.

فراموش نکنید که هنگام نمایه‌سازی، بر روی رایانه‌ی محلی‌تان، نهان‌سازی HTTP را در ``config/packages/framework.yaml`` غیرفعال کنید: اگر اینکار را نکنید، شما به جای کدتان، لایه‌ی نهان‌سازی HTTP در سیمفونی را نمایه‌سازی خواهید کرد.

برای به دست آوردن یک تصویر بهتر از کارایی اپلیکیشن‌تان در محیط عمل‌آوری، شما باید محیط «عمل‌آوری» را نیز نمایه‌سازی کنید. به صورت پیش‌فرض، محیط محلی شما از محیط «توسعه» استفاده می‌کند که سربار زیادی ایجاد می‌نماید (غالباً برای جمع‌آوری داده برای نوارابزار اشکال‌زدایی وب و نمایه‌ساز سیمفونی).

.. note::

    از آنجایی که محیط «عمل‌آوری» را نمایه‌سازی خواهیم کرد، نیازی به تغییر چیزی در پیکربندی نیست، زیرا در فصلی پیشین لایه‌ی نهان‌سازی HTTP سیمفونی را تنها برای محیط «توسعه» فعال کردیم.

.. index::
    single: Symfony CLI;server:prod

تعویض محیط رایانه‌ی محلی‌تان به محیط عمل‌آوری، از طریق تغییر متغیر محیط ``APP_ENV`` در فایل ``.env.local`` قابل انجام است:

.. code-block:: text
    :class: ignore

    APP_ENV=prod

یا می‌توانید از فرمان ``server:prod`` استفاده کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

فراموش نکنید که پس از پایان نمایه‌سازی، به محیط توسعه برگردید:

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

نمایه‌سازی منابع API
------------------------------------

.. index::
    single: Profiling;API

بهتر است نمایه‌سازی API، با CLI و از طریق ابزار Blackfire CLI که قبلاً نصب کردید، انجام شود:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

فرمان ``blackfire curl`` دقیقاً همان آرگمان‌ها و گزینه‌های `cURL`_  را می‌پذیرد.

مقایسه‌ی کارایی
------------------------------

در گام مربوط به «نهان‌سازی»، ما لایه‌ی نهان‌سازی را برای افزایش کارایی کدمان اضافه کردیم اما تأثیر این تغییر بر کارایی را اندازه‌گیری یا بررسی نکردیم. از آنجایی که همه‌ی ما در حدس‌زدن اینکه چه‌چیزی سریع و چه‌چیزی کند خواهد بود، بسیار بد هستیم، شما ممکن است در نهایت به وضعیتی برسید که برخی بهینه‌سازی‌ها در حقیقت اپلیکیشن‌تان را کندتر کند.

شما باید همیشه تأثیر هر بهینه‌سازی‌ای که انجام می‌دهید را با یک نمایه‌ساز اندازه بگیرید. Blackfire به کمک `ویژگی مقایسه‌اش`_، به صورت بصری آن را ساده‌تر می‌کند.

نوشتن آزمون‌های کارکردی جعبه‌سیاه
-----------------------------------------------------------------

.. index::
    single: Blackfire;Player

ما نحوه‌ی نوشتن آزمون‌های کارکردی در سیمفونی را دیده‌ایم. Blackfire می‌تواند برای نوشتن سناریوهای مرور‌کردن وب‌سایت استفاده شود که در موقع نیاز می‌توان آن را از طریق `Blackfire player`_ اجرا کرد. بیایید یک سناریو برای ارسال یک کامنت جدید بنویسیم که در محیط توسعه اعتبارسنجی آن از طریق پیوند رایانامه و در محیط عمل‌آوری از طریق مدیر انجام می‌شود.

یک فایل ``.blackfire.yaml`` با محتوای زیر ایجاد کنید:

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment[author] 'Fabien'
                param comment[email] 'me@example.com'
                param comment[text] 'Such a good conference!'
                param comment[photo] file(fake('simple_image', '/tmp', 400, 300, 'png', true, true), 'placeholder-image.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin')
                    expect status_code() == 302
                follow
                click link("Comments")
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

Blackfire player را بارگیری کنید تا قادر به اجرای سناریو به صورت محلی باشید:

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

سناریو را در محیط توسعه اجرا کنید:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

یا در محیط عمل‌آوری:

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

همچنین سناریوهای Blackfire می‌توانند برای هر درخواست نمایه‌سازی را راه‌اندازی کرده و با افزودن پرچم ``--blackfire``، آزمون‌های کارایی را اجرا کنند.

خودکارسازی بررسی‌های کارایی
-----------------------------------------------------

مدیریت کارایی تنها درباره‌ی بهبود کارایی کد موجود نیست و شامل بررسی اینکه روند کارایی نزولی نباشد نیز هست.

سناریوی نوشته‌شده در بخش قبلی می‌تواند به صورت خودکار در یک جریان‌کار ادغام مستمر (Continuous Integration) یا در محیط عمل‌آوری و بر اساس یک زمان‌بندی منظم اجرا گردد.

همچنین در Upsun این اجازه داده شده است که وقتی یک شاخه جدید ایجاد می‌کنید یا در محیط عمل‌آوری استقرار انجام می‌دهید، برای بررسی کارایی کد جدید به صورت خودکار  `سناریوها اجرا شوند`_.

.. sidebar:: بیشتر بدانید

    * `کتاب Blackfire: تشریح کارایی کد PHP`_؛

    * `آموزش تصویری Blackfire در SymfonyCasts`_.

.. _`Blackfire`: https://blackfire.io
.. _`ثبت‌نام کنید`: https://blackfire.io/signup
.. _`صفحه‌ی`: https://blackfire.io/my/settings/credentials
.. _`افزونه‌ی اختصاصی آن‌ها`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`ویژگی مقایسه‌اش`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`Blackfire player`: https://blackfire.io/player
.. _`سناریوها اجرا شوند`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`کتاب Blackfire: تشریح کارایی کد PHP`: https://blackfire.io/book
.. _`آموزش تصویری Blackfire در SymfonyCasts`: https://symfonycasts.com/screencast/blackfire
