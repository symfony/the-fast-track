تبني منهجية عمل
============================

إن التعليم ما هو إلا تكرار و إعادة لنفس العملية باستمرار. أعدكم بألا أقوم بذلك. عليك القيام برقصة صغيرة و تحفظ عملك عند الانتهاء من كل خطوة. ما هي الا ``Ctrl+S`` و لكن لموقع الكتروني

تطبيق استراتيجية من Git
----------------------------------------

.. index::
    single: Git;add
    single: Git;commit

لا تنسى ان تقوم بعملية commit للتغييرات التي قمت بها

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add some new feature'

يمكنك ان تقوم باضافة كل التغييرات بامان لان Symfony تدير ملف ``.gitignore`` بالنيابة عنك. كل package قد تقوم باضافة المزيد من التهيئات (configuration). الق نظرة على المكونات الحالية:

.. code-block:: text
    :caption: .gitignore
    :class: ignore
    :emphasize-lines: 1,9

    ###> symfony/framework-bundle ###
    /.env.local
    /.env.local.php
    /.env.*.local
    /config/secrets/prod/prod.decrypt.private.php
    /public/bundles/
    /var/
    /vendor/
    ###< symfony/framework-bundle ###

سلسلة الاحرف الغريبة ما هي الا علامات مضافة من قبل Symfony Flex لكي تحدد ما يجب ازالته في حالة الغاء واحدة من متطلبات تشغيل البرنامج (dependency). كما قلت لك سابقا، Symfony تقوم بكل الاعمال المضجرة، و ليس انت

من الجيد ان تقوم بعمل push لل repository الى خادم في مكان ما. GitHub ، GitLab او Bitbucket اختيارات جيدة.

.. note::

    اذا كنت تستخدم Upsun لنشر برنامجك، فسيكون لديك نسخة من ال repository لأن Upsun يستخدم Git خلف الكواليس عند استخدامك ل ``cloud:push``. و لكن لا يمكنك الاعتماد على repository الخاص بـ Upsun لانه فقط لخدمة النشر و ليس نسخة احتياطية.

النشر باستمرار على بيئة المنتج النهائي
-----------------------------------------------------------------------

.. index::
    single: Symfony CLI;cloud:push

من العادات الاخرى الجيدة هو ان تقوم بالنشر بشكل دوري. النشر في اخر كل خطوة هي سرعة جيدة:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:push
