من الصفر إلى المنتج النهائي
==================================================

أحب الحركة بشكل سريع. أريد لمشروعنا الصغير أن يظهر للنور في أسرع وقت ممكن. الأن اذا أمكن، حيث أننا لم نقم بتطوير أي شيء بعد، سنبدأ بإرفاق صفحة "تحت الإنشاء" بسيطة ولطيفة. ستحبها!

اقض بعض الوقت بحثا عن صورة "تحت الإنشاء" التقليدية المثالية المتحركة على شبكة الأنترنت. ها هي `الصورة`_ التي سأستخدمها:

.. image:: images/under-construction.gif
    :align: center

أخبرتك، سيكون الأمر به الكثير من المتعة.

تهيئة المشروع
-------------------------

أنشيء مشروع جديد باستخدام  أداة سطر الأوامر الخاصة بسيمفوني ``symfony`` التي قمنا بتثبيتها سويا من قبل:

.. code-block:: terminal

    $ symfony new guestbook --version=8.1 --php=8.5 --webapp --docker --upsun
    $ cd guestbook

هذا الأمر هو غلاف رقيق ل ``Composer`` لتسهيل انشاء مشاريع سيمفوني. يستخدم الأمر `هيكلية مشروع`_ تتضمن الحد الأدنى من التبعيات؛ مكونات سيمفوني المطلوبة تقريبا لكل مشروع: أداة تحكم طرفي Console Tool و تخطيط تجريدي لبروتوكول HTTP مطلوبة لبناء مشاريع الويب.

وبما أننا ننشئ تطبيق ويب متكامل المميزات، أضفنا بعض الخيارات التي ستسهل حياتنا:

* ``--webapp``: بشكل افتراضي، يُنشأ تطبيق بأقل قدر ممكن من التبعيات. لمعظم مشاريع الويب، يُنصح باستخدام حزمة ``webapp`` فوقه. فهي تحتوي على معظم الحزم اللازمة لتطبيقات الويب "الحديثة". تضيف حزمة ``webapp`` الكثير من حزم سيمفوني، بما في ذلك Symfony Messenger و PostgreSQL عبر Doctrine.

* ``--docker``: على جهازك المحلي، سنستخدم Docker لإدارة الخدمات مثل PostgreSQL. يُفعّل هذا الخيار Docker بحيث تضيف سيمفوني خدمات Docker تلقائيا بناءً على الحزم المطلوبة (خدمة PostgreSQL عند إضافة ORM أو ملتقط بريد عند إضافة Symfony Mailer على سبيل المثال).

* ``--upsun``: إذا أردت نشر مشروعك على Upsun، فإن هذا الخيار يولّد تلقائيا إعدادات Upsun معقولة. Upsun هي الطريقة المفضلة والأبسط لنشر بيئات سيمفوني للاختبار والتجهيز والإنتاج في السحابة.

اذا نظرت إلى مستودع GitHub لهيكلية المشروع، ستلاحظ أنها فارغة تقريبا. فقط ملف ``composer.json`` . لكن مجلد ``guestbook`` ملآن بالملفات. كيف يمكن أن يحدث هذا؟ الإجابة تقبع في حزمة ``symfony/flex``. سيمفوني فليكس Symfony Flex عباره عن إضافة لـ Composer تقوم بالتعلق في عملية التثبيت. عندما تكتشف حزمة لها *وصفه*، تقوم بتنفيذها.

نقطة الدخول الرئيسية لوصفة سيمفوني هي ملف بيان manifest file يصف العمليات التي نحتاج اجراءها لتسجيل الحزمة بشكل تلقائي في مشروع سيمفوني. يجب ألا تضطر إلى قراءة ملف README لتثبيت حزمة مع سيمفوني. الأتمتة Automation ميزة أساسية في سيمفوني.

بما أنه Git مثبت على الجهاز، قام ``symfony new`` بانشاء مستودع Git لنا وأضافت أول عملية التزام بالكود commit.

الق نظرة على تخطيط المجلد:

.. code-block:: text
    :class: ignore

    ├── bin/
    ├── composer.json
    ├── composer.lock
    ├── config/
    ├── public/
    ├── src/
    ├── symfony.lock
    ├── var/
    └── vendor/

مجلد ``bin/`` يحتوي نقطة سطر الأوامر الرئيسية: ``console``. ستستخدمها طوال الوقت.

مجلد ``config/`` مكون من عدة ملفات اعدادات أفتراضية و مكتمله. ملف لكل حزمة. نادرا ما ستلجأ لتغييرهم، الثقة في الافتراضيات غالبا ما تكون فكرة جيدة إلى حد ما.

مجلد ``public/`` هو المجلد الرئيسي للويب، و سكربت ``index.php`` هو نقطة الدخول الأساسية لكل موارد HTTP المتغيرة.

مجلد ``src/`` به كل الكود الذي ستكتبه؛ هناك ستقضي معظم وقتك. بشكل أساسي، كل ال Classes تحت هذا المجلد تستخدم ``App`` كمساحة اسم PHP namespace. إنه بيتك. الكود الذي يخصك. منطق مجال العمل Domain Logic الخاص بك. سيمفوني لديها القليل فيما يخصها هناك.

مجلد ``var/`` يحتوي التخزين المؤقت caches، سجلات الحركة logs و الملفات المتولدة أثناء عملية التشغيل runtime بواسطة التطبيق. يمكنك أن تتركها وشأنها. هو المجلد الوحيد الذي تحتاج له أن يكون قابلا للكتابة في التشغيل النهائي production.

مجلد ``vendor/`` يحتوي كل الحزم المثبتة بواسطة Composer، متضمنة سيمفوني نفسها. هذا هو سلاحنا السري لزيادة الانتاجية. دعنا لا نعيد اختراع العجلة. ستعتمد على مكتبات برمجية متواجدة بالفعل لانجاز العمل الصعب. المجلد يتم ادارته عن طريق Composer. لا تلمسه أبدا.

هذا كل ما تحتاج إلى معرفته الآن.

إنشاء بعض الموارد العامة
---------------------------------------------

أي شيء تحت مجلد ``public/`` يمكن الوصول إليه عبر المتصفح. على سبيل المثال، إذا نقلت ملف الصورة المتحركة (قم بتسميتها ``under-construction.gif``) إلى مجلد ``public/images/``، ستكون متاحة على رابط لمحدد موقع الموارد URL مثل ``https://localhost/images/under-construction.gif``.

تنزيل الصورة المتحركة الخاصة بي هنا:

.. code-block:: terminal

    $ mkdir public/images/
    $ php -r "copy('https://clipartmag.com/images/website-under-construction-image-6.gif', 'public/images/under-construction.gif');"

إطلاق خادم ويب محلي
-----------------------------------

.. index::
    single: Symfony CLI;server:start

سطر الأوامر المرفق مع ``symfony`` يأتي مع خادم ويب محسن لشغل التطوير. لن تتفاجأ إذا أخبرتك أنه يعمل بشكل رائع مع سيمفوني. لا تستخدمه في المنتج النهائي رغم ذلك.

من مجلد المشروع، قم ببدء خادم الويب في الخلفية (استخدم العلامة ``-d``):

.. code-block:: terminal

    $ symfony server:start -d

يبدأ الخادم في العمل باستخدام أول منفذ متاح بداية من 8000. للوصول السريع، قم بفتح الموقع في المتصفح باستخدام سطر الأوامر:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local

متصفحك المفضل يجب أن يظهر الآن و يقوم بفتح تبويب جديد الذي بدوره سيعرض شيئا شبيها لما يلي:

.. figure:: screenshots/symfony-greetings.png
    :alt: /
    :align: center
    :figclass: with-browser

.. tip::

    لاكتشاف المشكلات، شغل ``symfony server:log``؛ تقوم بعرض سجلات الحركة الأخيرة من خادم الويب و PHP وتطبيقك.

تصفح إلى ``/images/under-construction.gif``. هل تبدو لك مثل هذا؟

.. figure:: screenshots/under-construction-web.png
    :alt: /images/under-construction.gif
    :align: center
    :figclass: with-browser

.. index::
    single: Git;add
    single: Git;commit

راض ؟ دعنا نسجل التزام عملنا commit our work:

.. code-block:: terminal
    :class: ignore

    $ git add public/images
    $ git commit -m'Add the under construction image'

الإعداد للمنتج النهائي
------------------------------------------

.. index::
    single: Upsun;Initialization

ماذا عن رفع المنتج النهائي للنشر؟ أعرف، ليس لدينا حتى الأن صفحة HTML مناسبة للترحيب بالمستخدمين. لكن إذا تمكنا من عرض صورة "تحت الإنشاء" على خادم المنتج النهائي فستكون خطوة عظيمة للأمام. وأنت تعرف المقولة: *انشر عملك بشكل مبكر ودائم*

يمكنك استضافة هذا التطبيق على أي مقدم خدمة يدعم PHP .. مما يعني تقريبا كل مقدمي الخدمات. افحص بعض الأشياء رغم ذلك: نريد أخر نسخة من PHP وإمكانية استضافة خدمات مثل قاعدة بيانات، قائمة الانتظار queue، وبعض الخدمات الأخرى.

لقد حددت اختياري، سيكون `Upsun`_. تقدم كل ما نحتاجه و تساعد في تمويل عملية التطوير الخاصة بسيمفوني.

.. index::
    single: Symfony CLI;project:init

بما أننا استخدمنا الخيار ``--upsun`` عند إنشاء المشروع، فقد تمت تهيئة Upsun بالفعل بملف الإعدادات الوحيد الذي تحتاجه، وهو ``.upsun/config.yaml``.

الانتقال إلى الإنتاج النهائي
-----------------------------------------------------

.. index::
    single: Symfony CLI;cloud:project:create
    single: Symfony CLI;cloud:push

وقت نشر المنتج؟

أنشيء مشروع Upsun بعيد جديد:

.. code-block:: terminal

    $ symfony cloud:project:create --title="Guestbook" --plan=development

هذا الأمر يفعل الكثير:

* المرة الأولى التي تنفذ فيها هذا الأمر، قم بعمل المصادقة مع اعدادات دخولك لـ Upsun اذا لم تقم بذلك من قبل.

* تقوم بتوفير مشروع جديد على Upsun (تحصل على 30 يوما *مجانا* على أول مشروع تنشئه).

بعد ذلك، ارفع deploy:

.. code-block:: terminal

    $ symfony cloud:push

يتم رفع الكود عن طريق دفع pushing إلى مستودع Git. بعد تنفيذ الأمر، يحصل المشروع على اسم نطاق خاص يمكنك استخدامه للوصول له.

.. index::
    single: Symfony CLI;cloud:url

تأكد من أن كل شيء يعمل بشكل جيد:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

يجب أن تحصل على 404، لكن التصفح إلى ``/images/under-construction.gif`` سيكشف عن ما تم عمله.

لاحظ أنك لم تحصل على صفحة سيمفوني الافتراضية الجميلة على Upsun. لماذا؟ ستتعلم قريبا أن سيمفوني تدعم أكثر من بيئة وقامت Upsun بشكل تلقائي بالنشر مستخدمة بيئة المنتج النهائي.

.. index::
    single: Symfony CLI;cloud:project:delete

.. tip::

    اذا أردت حذف المشروع من Upsun، يمكنك استخدام أمر ``cloud:project:delete``.

.. sidebar:: الذهاب أبعد من ذلك

    * مستودعات `وصفات سيمفوني الرسمية`_ وكذلك `الوصفات المساهمة من المجتمع`_ حيث يمكنك ارسال وصفاتك الخاصة؛

    * `خادم الويب المحلي الخاص بسيمفوني`_؛

    * `وثائق Upsun`_.

.. _`الصورة`: https://clipartmag.com/images/website-under-construction-image-6.gif
.. _`هيكلية مشروع`: https://github.com/symfony/skeleton
.. _`Upsun`: https://upsun.com/symfony/?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
.. _`وصفات سيمفوني الرسمية`: https://github.com/symfony/recipes
.. _`الوصفات المساهمة من المجتمع`: https://github.com/symfony/recipes-contrib
.. _`خادم الويب المحلي الخاص بسيمفوني`: https://symfony.com/doc/current/setup/symfony_server.html
.. _`وثائق Upsun`: https://developer.upsun.com/docs/get-started/stacks/symfony/index?utm_source=symfony-cloud-sign-up&utm_medium=backlink&utm_campaign=Symfony-Cloud-sign-up&utm_content=symfony-book
