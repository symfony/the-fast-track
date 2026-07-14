إكتشاف بنية سيمفوني الداخلية
=====================================================

.. index::
    single: Blackfire
    single: Debugging
    single: Internals

لقد تم استخدام Symfony لتطوير تطبيق قوي لفترة طويلة الآن ، ولكن معظم التعليمات البرمجية التي ينفذها التطبيق تأتي من Symfony. بضع مئات من سطور الكود مقابل آلاف سطور الكود.

أحب أن أفهم كيف تعمل الأشياء وراء الكواليس. لقد كنت مفتونًا دائمًا بالأدوات التي تساعدني في فهم كيفية عمل الأشياء. في المرة الأولى التي استخدمت فيها مصحح الأخطاء خطوة بخطوة أو المرة الأولى التي اكتشفت فيها ``ptrace`` ذكريات سحرية.

هل ترغب في فهم كيفية عمل Symfony بشكل أفضل؟ حان الوقت للتفكير في كيفية جعل Symfony علامة التطبيق الخاص بك. بدلاً من وصف كيفية معالجة Symfony لطلب HTTP من منظور نظري ، والذي سيكون مملاً للغاية ، فإننا سنستخدم Blackfire للحصول على بعض العروض المرئية واستخدامها لاكتشاف بعض الموضوعات المتقدمة.

فهم بنية سيمفوني الداخلية عن طريق Blackfire
-----------------------------------------------------------------------

أنت تعلم بالفعل أن جميع طلبات HTTP يتم تقديمها بواسطة نقطة إدخال واحدة: ملف `public / index.php`. لكن ماذا سيحدث بعد ذلك؟ كيف تسمى وحدات التحكم؟

دعنا نوصِّف الصفحة الرئيسية للغة الإنجليزية في الإنتاج باستخدام Blackfire عبر ملحق مستعرض Blackfire:

.. code-block:: terminal
    :class: ignore

    $ symfony remote:open

أو مباشرة عبر واجهة الأوامر:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony cloud:env:url --pipe --primary`en/

انتقل إلى عرض "الجدول الزمني" لملف التعريف ، سترى شيئًا مشابهًا لما يلي:

.. figure:: images/blackfire-homepage-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

من الجدول الزمني ، قم بالتمرير فوق الأشرطة الملونة للحصول على مزيد من المعلومات حول كل مكالمة ؛ سوف تتعلم الكثير عن كيفية عمل Symfony:

* نقطة الدخول الرئيسية هي ``public / index.php``؛

* يعالج الأسلوب ``Kernel::handle()`` الطلب ؛

* يطلق عليه "HttpKernel" الذي يرسل بعض الأحداث ؛

* الحدث الأول هو  ``RequestEvent`` ؛

* يتم استدعاء الأسلوب `` ControllerResolver :: getController () `` لتحديد وحدة التحكم التي يجب استدعاؤها لعنوان URL الوارد ؛

* يتم استدعاء الأسلوب `` ControllerResolver :: getArguments () `` لتحديد الوسائط التي سيتم تمريرها إلى وحدة التحكم (يسمى المحول البارز) ؛

* يتم استدعاء الأسلوب `` ConferenceController :: index () `` ويتم تنفيذ معظم التعليمات البرمجية الخاصة بنا بواسطة هذه المكالمة ؛

* تحصل الطريقة ``ConferenceRepository::findAll()`` على جميع المؤتمرات من قاعدة البيانات (لاحظ الاتصال بقاعدة البيانات عبر ``PDO::__construct()``) ؛

* الأسلوب ``Twig\Environment::render()`` يعرض القالب ؛

* تسمى "ResponseEvent" و "FinishRequestEvent" ، ويتم إرسالهما ، لكن يبدو أنه لا يوجد مستمعين مسجلين بالفعل حيث يبدو أنهما سريعان في التنفيذ.

الخط الزمني هو وسيلة رائعة لفهم كيفية عمل بعض التعليمات البرمجية ؛ وهو أمر مفيد للغاية عندما تحصل على مشروع قام بتطويره شخص آخر.

الآن ، قم بملف تعريف الصفحة نفسها من الجهاز المحلي في بيئة التطوير:

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/

افتح ملف التعريف. يجب إعادة توجيهك إلى عرض الرسم البياني للمكالمات حيث أن الطلب كان سريعًا بالفعل وسيكون الجدول الزمني فارغًا تمامًا:

.. figure:: images/blackfire-homepage-cached-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

هل تفهم ما الذي يحدث؟ يتم تمكين ذاكرة التخزين المؤقت لـ HTTP، وبالتالي، فإننا نقوم بتوصيف طبقة ذاكرة التخزين المؤقت HTTP Symfony. نظرًا لأن الصفحة في ذاكرة التخزين المؤقت، ``HttpCache\Store::restoreResponse()`` يحصل على استجابة HTTP من ذاكرة التخزين المؤقت الخاصة به ولا يتم استدعاء وحدة التحكم أبدًا.

تعطيل طبقة ذاكرة التخزين المؤقت في `public / index.php`` كما فعلنا في الخطوة السابقة وحاول مرة أخرى. يمكنك أن ترى على الفور أن الملف الشخصي يبدو مختلفًا تمامًا:

.. figure:: images/blackfire-homepage-dev.png
    :alt: /
    :align: center
    :figclass: with-browser

الاختلافات الرئيسية هي ما يلي:

* تأخذ "TerminateEvent" ، التي لم تكن مرئية في الإنتاج ، نسبة مئوية كبيرة من وقت التنفيذ ؛ إذا نظرنا عن كثب ، يمكنك أن ترى أن هذا هو الحدث المسؤول عن تخزين بيانات ملفات تعريف Symfony التي تم جمعها أثناء الطلب ؛

* ضمن `` ConferenceController :: index () `` call ، لاحظ الأسلوب `` SubRequestHandler :: handle () `` الذي يعرض ESI (لهذا السبب لدينا مكالمتان ل `` Profiler :: saveProfile () `` ، ، واحد للطلب الرئيسي وواحد لل ESI).

استكشف الجدول الزمني لمعرفة المزيد ؛ قم بالتبديل إلى عرض الرسم البياني للمكالمات للحصول على تمثيل مختلف لنفس البيانات.

كما اكتشفنا للتو ، الكود الذي تم تنفيذه في التطوير والإنتاج مختلف تمامًا. بيئة التطوير أبطأ حيث يحاول منشئ ملفات التعريف Symfony جمع العديد من البيانات لتخفيف مشاكل التصحيح. لهذا السبب يجب عليك دائمًا التعريف ببيئة الإنتاج ، حتى محليًا.

بعض التجارب المثيرة للاهتمام: ملف تعريف صفحة خطأ ، أو ملف تعريف الصفحة `` / `` (التي هي إعادة توجيه) ، أو مورد API. سيخبرك كل ملف تعريف أكثر قليلاً حول كيفية عمل Symfony ، ما هي الفئة / الأساليب التي يتم تسميتها ، وما يكلف غاليا تشغيلة وماهو بسيط.

بإستخدام إضافة تصحيح Blackfire
------------------------------------------------

.. index::
    single: Blackfire;Debug Addon

بشكل افتراضي ، يزيل Blackfire جميع استدعاءات الطرق غير المهمة بما يكفي لتجنب وجود حمولات كبيرة ورسوم بيانية كبيرة. عند استخدام Blackfire كأداة لتصحيح الأخطاء ، من الأفضل الاحتفاظ بجميع المكالمات. يتم توفير هذا بواسطة الملحق التصحيح.

من سطر الأوامر ، استخدم علامة `- debug``:

.. code-block:: terminal
    :class: ignore

    $ blackfire --debug curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`en/
    $ blackfire --debug curl `symfony cloud:env:url --pipe --primary`en/

.. index::
    single: .env.local.prod

في الإنتاج ، سترى على سبيل المثال تحميل ملف باسم ``.env.local.php``:

.. figure:: images/blackfire-env-local-prod.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Composer;Optimizations
    single: Composer;Autoloader
    single: Autoloader

حيث أنها لا تأتي من؟ تقوم Upsun ببعض التحسينات عند نشر تطبيق Symfony مثل تحسين التحميل التلقائي للملحن (`` - --optimize-autoloader --apcu-autoloader --classmap-Authoritative``). كما أنه يحسن متغيرات البيئة المعرفة في ملف `` .env`` (لتجنب تحليل الملف لكل طلب) عن طريق إنشاء ملف `` .env.local.prod`` php``:

.. code-block:: terminal
    :class: ignore

    $ symfony run composer dump-env prod

Blackfire هي أداة قوية للغاية تساعد على فهم كيفية تنفيذ تعليمات البرمجية بواسطة PHP. تحسين الأداء  هو مجرد طريقة لاستخدام منشئ ملفات التعريف (profiler).

استخدام Step Debugger مع Xdebug
----------------------------------------

.. index::
    single: Xdebug
    single: Debugger

تتيح المخططات الزمنية لـ Blackfire والرسوم البيانية للمطورين تصور الملفات و الدوال المنفذة بواسطة محرك PHP لفهم أفضل لتكوين بناء وشيفرة المشروع.

هناك طريقة أخرى لمتابعة تنفيذ التعليمات البرمجية وهي استخدام **step debugger** مثل `Xdebug`_ يسمح مصحح الأخطاء للمطورين بالتصفح بشكل تفاعلي عبر رمز مشروع PHP لتصحيح أخطاء التحكم في التدفق وفحص هياكل البيانات. من المفيد جدًا تصحيح السلوكيات غير المتوقعة واستبدال تقنية التصحيح الشائعة "var_dump () / exit ()".

أولاً ، قم بتثبيت ``xdebug`` امتداد PHP . تحقق من تثبيته عن طريق تشغيل الأمر التالي:

.. code-block:: terminal

    $ symfony php -v

يجب أن ترى Xdebug في الإخراج:

.. code-block:: text
    :emphasize-lines: 5
    :class: ignore

    PHP 8.0.1 (cli) (built: Jan 13 2021 08:22:35) ( NTS )
    Copyright (c) The PHP Group
    Zend Engine v4.0.1, Copyright (c) Zend Technologies
        with Zend OPcache v8.0.1, Copyright (c), by Zend Technologies
        with Xdebug v3.0.2, Copyright (c) 2002-2021, by Derick Rethans
        with blackfire v1.49.0~linux-x64-non_zts80, https://blackfire.io, by Blackfire

يمكنك أيضًا التحقق من تمكين Xdebug لـ PHP-FPM بالانتقال إلى المتصفح والنقر على رابط "View phpinfo ()" عند التمرير فوق شعار سيمفوني لشريط أدوات تصحيح أخطاء الويب:

.. figure:: screenshots/phpinfo.png
    :alt: /
    :align: center
    :figclass: with-browser

الآن ، قم بتمكين وضع ``debug`` في Xdebug:

.. code-block:: ini
    :caption: php.ini
    :class: ignore

    [xdebug]
    xdebug.mode=debug
    xdebug.start_with_request=yes

بشكل افتراضي ، يرسل Xdebug البيانات إلى المنفذ 9003 للمضيف المحلي.

يمكن إجراء تشغيل Xdebug بعدة طرق ، ولكن أسهلها هو استخدام Xdebug من IDE الخاص بك. في هذا الفصل ، سنستخدم Visual Studio Code لشرح كيفية عمله. قم بتثبيت `PHP Debug`_ من خلال تشغيل ميزة "Quick Open" (`` Ctrl + P '') ، الصق الأمر التالي ، واضغط على Enter:

.. code-block:: text
    :class: ignore

    ext install felixfbecker.php-debug

قم بإنشاء ملف التكوين التالي:

.. code-block:: json
    :caption: .vscode/launch.json
    :emphasize-lines: 8,16
    :class: ignore

    {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Listen for XDebug",
                "type": "php",
                "request": "launch",
                "port": 9003
            },
            {
                "name": "Launch currently open script",
                "type": "php",
                "request": "launch",
                "program": "${file}",
                "cwd": "${fileDirname}",
                "port": 9003
            }
        ]
    }

من Visual Studio Code وأثناء وجودك في دليل المشروع ، انتقل إلى مصحح الأخطاء وانقر على زر التشغيل الأخضر المسمى "Listen for Xdebug":

.. figure:: images/vs-xdebug-run.png
    :align: center

إذا ذهبت إلى المتصفح وقمت بالتحديث ، يجب أن يأخذ IDE التركيز تلقائيًا ، مما يعني أن جلسة التصحيح جاهزة. بشكل افتراضي ، كل شيء هو نقطة توقف ، لذلك يتوقف التنفيذ عند أول تعليمات. بعد ذلك ، يعود الأمر إليك لفحص المتغيرات الحالية ، وتخطي الكود ، والدخول إلى الكود ، ...

عند تصحيح الأخطاء ، يمكنك إلغاء تحديد نقطة توقف "كل شيء" وتعيين نقاط التوقف في التعليمات البرمجية الخاصة بك.

إذا كنت جديدًا في استخدام أدوات تصحيح الأخطاء ، فاقرأ `excellent tutorial for Visual Studio Code`_, والذي يشرح كل شيء بصريًا.

.. sidebar:: الذهاب أبعد من ذلك

    * `The Xdebug Step Debugging docs`_;

    * `Debugging with Visual Studio Code`_.

.. _`Xdebug`: https://xdebug.org
.. _`PHP Debug`: https://marketplace.visualstudio.com/items?itemName=felixfbecker.php-debug
.. _`The Xdebug Step Debugging docs`: https://xdebug.org/docs/step_debug
.. _`excellent tutorial for Visual Studio Code`: https://code.visualstudio.com/Docs/editor/debugging
.. _`Debugging with Visual Studio Code`: https://code.visualstudio.com/Docs/editor/debugging
