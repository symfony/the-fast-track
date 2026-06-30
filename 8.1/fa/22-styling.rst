طراحی ظاهر رابط کاربری
==================================================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

ما هیچ زمانی را صرف طراحی رابط کاربری نکرده‌ایم. برای طراحی همچون یک حرفه‌ای، از یک پشته‌ی مدرن مبتنی بر *AssetMapper* استفاده می‌کنیم، یعنی کامپوننت سیمفونی که از همان نخستین گام این کتاب، دارایی‌های (assets) ما را مدیریت کرده است.

AssetMapper استانداردهای مدرن وب را در آغوش می‌گیرد: فایل‌های JavaScript و CSS همان‌گونه که هستند سرو می‌شوند و با یک *importmap* به هم متصل می‌گردند، که به مرورگر اجازه می‌دهد *ES modules* بومی را مستقیماً بارگیری کند. بدون bundler، بدون گام ساخت (build)، بدون Node.js.

به فایل ``importmap.php`` در ریشه‌ی پروژه نگاهی بیندازید: این فایل بسته‌های JavaScript استفاده‌شده توسط اپلیکیشن را توصیف می‌کند. تابع ``importmap()`` در Twig که در ``templates/base.html.twig`` فراخوانی می‌شود، آن‌ها را به مرورگر عرضه می‌کند.

بهره‌گیری از Bootstrap
------------------------------------

.. index::
    single: Bootstrap

برای شروع با مقادیر پیش‌فرض خوب و ساخت یک وب‌سایت واکنش‌گرا (responsive)، یک چارچوب CSS مانند `Bootstrap`_ می‌تواند راه درازی را برایتان هموار کند. آن را به عنوان یک بسته‌ی importmap نصب کنید:

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

این فرمان بسته را در ``importmap.php`` ثبت کرده و آن را (به همراه وابستگی ``@popperjs/core`` آن) در ``assets/vendor/`` بارگیری می‌کند؛ اپلیکیشن در زمان اجرا به یک CDN وابسته نیست.

Bootstrap را در مدخل اصلی JavaScript وارد (import) کنید (ما همچنین پیغام خوش‌آمدگویی پیش‌فرض را پاک کرده‌ایم):

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -5,6 +5,6 @@ import './stimulus_bootstrap.js';
      * This file will be included onto the page via the importmap() Twig function,
      * which should already be in your base.html.twig.
      */
    +import 'bootstrap';
    +import 'bootstrap/dist/css/bootstrap.min.css';
     import './styles/app.css';
    -
    -console.log('This log comes from assets/app.js - welcome to AssetMapper! 🎉');

توجه کنید که ``app.css`` *پس از* استایل‌های Bootstrap وارد می‌شود تا سفارشی‌سازی‌های ما برنده شوند.

سیستم فرم سیمفونی به صورت بومی و با یک تم ویژه از Bootstrap پشتیبانی می‌کند، آن را فعال کنید:

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

طراحی ظاهر HTML
----------------------------

ما اکنون آماده‌ی طراحی ظاهر اپلیکیشن هستیم. آرشیو موجود در ریشه‌ی پروژه را بارگیری و باز کنید:

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

به قالب‌ها نگاهی بیندازید، شاید یکی دو ترفند درباره‌ی Twig یاد بگیرید.

سرو کردن دارایی‌ها (Assets)
---------------------------------------------

.. index::
    single: AssetMapper;asset-map:compile

چیزی برای ساختن وجود ندارد: یک صفحه را تازه‌سازی کنید و تغییرات زنده هستند. در محیط توسعه، AssetMapper فایل‌های دارایی را مستقیماً سرو می‌کند.

زمانی را صرف کشف تغییرات بصری کنید. به طراحی جدید در یک مرورگر نگاهی بیندازید.

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

فرم ورود تولیدشده نیز اکنون طراحی ظاهری دارد، زیرا باندل Maker به صورت پیش‌فرض از کلاس‌های CSS مربوط به Bootstrap استفاده می‌کند:

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

برای محیط عمل‌آوری، Upsun به صورت خودکار فرمان ``asset-map:compile`` را در طول مرحله‌ی build اجرا می‌کند: تمام دارایی‌ها به همراه یک hash نسخه در نام فایل‌هایشان به ``public/assets/`` کپی می‌شوند، که نهان‌سازی HTTP امن و بلندمدت را ممکن می‌سازد.

.. sidebar:: بیشتر بدانید

    * `مستندات کامپوننت AssetMapper`_؛

    * `مشخصات فنی importmap`_؛

    * `مستندات Bootstrap`_.

.. _`Bootstrap`: https://getbootstrap.com/
.. _`مستندات کامپوننت AssetMapper`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`مشخصات فنی importmap`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`مستندات Bootstrap`: https://getbootstrap.com/docs/
