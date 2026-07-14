إستكشاف الأخطاء وإصلاحها
==============================================

إعداد المشروع يتطلب أيضاً وجود الأدوات المناسبة لإصلاح المشكلات. ولحسن الحظ، فإن العديد من المساعدات اللطيفة مضمنة بالفعل كجزء من حزمة ``webapp``.

استكشاف ادوات تصحيح الاخطاء الخاصة بسيمفوني
---------------------------------------------------------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

في البداية، يُعد محلل سيمفوني منقذاً للوقت عندما تريد معرفة السبب الرئيسي لمشكلة معينة.

لو ألقيت نظرة علي الصفحة الرئيسية، يجب أن تري شريط ادوات في أسفل الشاشة:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

أول شئ من الممكن أن تلاحظه هو الـ **404** بالاحمر. تذكر بأن هذه الصفحة هي عنصر نائب لاننا لم نعرف الصفحة الرئيسية بعد. حتي لو أن الصفحة المبدئية التي تُرحبك تبدو جميلة، تظل صفحة خطأ. لذا فإن رمز حالة الـ HTTP الصحيح هو 404 وليس 200. بفضل شريط ادوات التصحيح، لديك المعلومات علي الفور.

لو قمت بالضغط علي علامة التعجب الصغيرة، سوف تحصل علي رسالة الخطأ "الحقيقية" كجزء من التجسيلات الموجودة في أداة تعريف (محلل) سيمفوني (Symfony profiler). إن كنت تريد ان تري حزمة التعقب، إضغط علي رابط "الاعتراض" (Exception) علي القائمة اليسري.

كلما كان هناك مشكلة في الرمز البرمجي الخاص بك، سوف تري صفحة اعتراض (استثناء) مثل التالية التي تعطيك كل شئ تحتاجة حتي تفهم المشكلة ومن اين تأتي:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

خذ بعض من الوقت لاستكشاف المعلومات الموجودة داخل محلل سيمفوني (Symfony profiler) عن طريق الضغط بالجوار.

.. index::
    single: Symfony CLI;server:log

التجسيلات مفيدة أيضاً في جلسات استقصاء وتصحيح الاخطاء. يمتلك سيمفوني علي أمر مناسب لتعقب (طباعة tail) كل التسجيلات (من الخادم، PHP، وتطبيقك):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

لنقوم بتجربة صغيرة. قم بفتح ``public/index.php`` وعطل او اكسر كود PHP الموجود هناك (علي سبيل المثال قم بإضافة كلمة foobar في الوسط). قم بتحديث الصفحة في المتصفح وراقب مجري التسجيلات:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

النتائج (المُخرجات) ملونة بشكل جميل لجذب انتباهك للاخطاء.

فهم بيئات عمل سيمفوني
---------------------------------------

.. index::
    single: Symfony Environments

بما أن محلل سيمفوني مفيد فقط أثناء عملية التطوير، نريد أن نتجنب تنصيبه (تسطيبه) في الانتاجية. بشكل افتراضي، ثبّتت سيمفوني المحلل تلقائياً فقط لبيئتي ``dev`` و ``test``.

يدعم سيمفوني مفهوم *بيئات العمل* (*environments*). بشكل افتراضي، تقوم بدعم ثلاثة، ولكنك يمكنك إضافة العدد الذي تريده: ``dev``، ``prod``، و ``test``. كل البيئات تشارك نفس الرمز البرمجي، ولكنهم يقدمون *إعدادات* (*configurations*) مختلفة.

فعلى سبيل المثال، جميع أدوات تصحيح الأخطاء متاحة في بيئة عمل ``dev``. ولكن في ال ``prod``، تم تحسين التطبيق من أجل الاداء.

التبديل من بيئة عمل الي أخري يمكن ان يتم عن طريق تغير متغير بيئة العمل ``APP_ENV``.

عندما تقوم بالنشر على Upsun، يتم تغير بيئة العمل (المخزنة في ``APP_ENV``) بشكل تلقائي الي ``prod``.

إدارة إعدادات بيئة عمل
-----------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

يمكن أن يتم ضبط متغير ``APP_ENV`` عن طريق استخدام متغيرات بيئة عمل "حقيقية" في شاشة الاومر الخاصة بيك:

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

استخدام متغيرات بيئة عمل حقيقية تعتبر افضل طريقة لتعيين قيم مثل ``APP_ENV`` علي خوادم الانتاجية. ولكن علي اجهزة التطوير، قد يكون الاضطرار لتعين العديد من متغيرات بيئة العمل امراً مرهقاً. وعوضاً عن ذلك، قم بتعريفهم في ملف الـ ``.env``.

بشكل منطقي تم إنشاء ملف الـ ``.env`` لكَ بشكل اوتماتيكي عندم أُنشئ المشروع.

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    يمكن لأي حزمة اضافة العديد من متغيرات بيئة العمل لهذا الملف بفضل وصفتهم التي تُستخدم بواسطة سيمفوني فليكس (Symfony Flex).

يتم الزام ملف الـ ``.env`` للمستودع (repository) ويصف القيم *المبدئية* من الانتاجية. يمكنك تجاوز هذه القيم عن طريق إنشاء ملف ``.env.local``. يجب ألا يتم الزام او رفع هذا الملف لذلك يقوم ملف الـ``.gitignore`` بتجاهله مسبقاً.

لا تخزّن ابداً قيم سرية او حساسة في هذه الملفات. سنري كيفية ادارة الاسرار في خطوة أخري.

إعدادات إطار عملك
--------------------------------

في بيئة التطوير، عندما يحدث استثناء او اعتراض، يقوم سيمفوني بعرض صفحة تحتوي علي رسالة هذا الاعتراض ومسار تتبعه. عند عرض مسار ملف، يقوم بإضافة رابط يقوم بفتح المف عند السطر الصحيح في الـ IDE المفضل لك. للاستفادة من هذه الميزة، قم بإعداد الـ IDE الخاص بك. يدعم سيموفني العديد من الـ IDEs خارج الصندوق؛ فأنا استخدم Visual Studio Code من اجل هذا المشروع:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

لا تقتصر الملفات المرتبطة علي الاستثناءات. علي سبيل المثال، المتحكم (وحدة التحكم) في شريط ادوات استقصاء وتصحيح الاخطاء يصبح قابل للضغط بعد اعداد الـIDE.

تصحيح أخطاء الإنتاج
------------------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:logs
    single: Symfony CLI;cloud:ssh

إستقصاء وتصحيح اخطاء خوادم الانتاجية أكثر صعوبة دائماً. ليس لديك حق الوصول لمحلل سيموفني علي سبيل المثال. السجلات اقل طولاً (verbose). ولكن طباعة او تذيل (tailing) السجلات ممكن:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --tail

يمكنك حتي الاتصال بواسطة SSH علي حاوية الويب:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

لا تقلق، فلا يمكنك كسر أي شئ بسهولة. اغلب نظام الملفات للقراءة فقط. لن يمكنك اجراء تصحيح سريع ومهم في الانتاجية. ولكن سوف تتعلم طريقة افضل لاحقا في الكتاب.

.. sidebar:: الذهاب أبعد من ذلك

    * `SymfonyCasts Environments and Config Files tutorial`_؛

    * `SymfonyCasts Environment Variables tutorial`_؛

    * `SymfonyCasts Web Debug Toolbar and Profiler tutorial`_؛

    * `Managing multiple .env files`_ in Symfony applications.

.. _`SymfonyCasts Environments and Config Files tutorial`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`SymfonyCasts Environment Variables tutorial`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`SymfonyCasts Web Debug Toolbar and Profiler tutorial`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`Managing multiple .env files`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
