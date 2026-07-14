تأمين الواجهة الخلفية للمدير
=====================================================

.. index::
    single: Components;Security
    single: Security

يجب أن تكون الواجهة الخلفية للمدير قابلة للوصول فقط من قبل الأشخاص الموثوق بهم. يمكن تأمين هذا الجزء من الموقع باستخدام Symfony SecurityComponent.

تحديد كيان (Entity) المستخدم
---------------------------------------------

حتى إذا لم يتمكن الحاضرون من إنشاء حساباتهم الخاصة على الموقع، فسننشئ نظام مصادقة كامل الوظائف للمدير. لذلك سيكون لدينا مستخدم واحد فقط ، مدير الموقع.

الخطوة الأولى هي تحديد كيان `` المستخدم ''. لتجنب أي ارتباك ، دعنا نسميها `` Admin '' بدلاً من ذلك.

لدمج كيان "المدير / Admin'' مع نظام مصادقة لSymfony Security ، نحتاج إلى اتباع بعض المتطلبات. على سبيل المثال ، يحتاج إلى خاصية "كلمة المرور/ password''.

.. index::
    single: Command;make:user

استخدم الأمر المخصص "make: user '' لإنشاء كيان  Entity "المدير/Admin '' بدلاً من الكيان التقليدي "make: entity'':

.. code-block:: terminal
    :class: answers(yes||username||yes)

    $ symfony console make:user Admin

أجب عن الأسئلة التفاعلية: نريد استخدام Doctrine لتخزين المديرين (" نعم / yes  '') ، استخدم "اسم المستخدم / username ' كإسم فريد للمديرين، وسيكون لكل مستخدم كلمة مرور / password  (" نعم / yes '') .

يحتوي الفصل الذي تم إنشاؤه على طرق مثل " getRoles() "  و '' eraseCredentials() `` ، وبعض الأنواع الأخرى التي يحتاجها نظام مصادقة Symfony.

إذا كنت ترغب في إضافة المزيد من الخصائص إلى المستخدم " Admin '' ، فاستخدم " make:entity ''.

بالإضافة إلى إنشاء كيان " المدير '' ، قام الأمر أيضًا بتحديث إعداد الأمان (The security configuration) لتوصيل الكيان بنظام المصادقة:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 11,12,20

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -5,14 +5,18 @@ security:
             Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'
         # https://symfony.com/doc/current/security.html#loading-the-user-the-user-provider
         providers:
    -        users_in_memory: { memory: null }
    +        # used to reload user from session & other features (e.g. switch_user)
    +        app_user_provider:
    +            entity:
    +                class: App\Entity\Admin
    +                property: username
         firewalls:
             dev:
                 pattern: ^/(_(profiler|wdt)|css|images|js)/
                 security: false
             main:
                 lazy: true
    -            provider: users_in_memory
    +            provider: app_user_provider

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#the-firewall

نسمح لـ Symfony باختيار أفضل خوارزمية ممكنة لتجزئة كلمات المرور passwords (التي ستتطور بمرور الوقت).

حان الوقت لإنشاء ترحيل (migration) وترحيل قاعدة البيانات:

.. code-block:: terminal

    $ symfony console make:migration
    $ symfony console doctrine:migrations:migrate -n

إنشاء كلمة مرور password للمستخدم المدير Admin
-------------------------------------------------------------------------

.. index::
    single: Security;Password Hashes

لن نطور نظامًا مخصصًا لإنشاء حسابات المدير. مرة أخرى ، سيكون لدينا مدير واحد فقط. سيكون تسجيل الدخول هو " admin '' ونحن بحاجة إلى توليد تجزئة كلمة المرور password .

.. index::
    single: Command;security:hash-password

اختر ما تريده ككلمة مرور وقم بتشغيل الأمر التالي لتوليد تجزئة كلمة المرور:

.. code-block:: terminal
    :class: answers(admin)

    $ symfony console security:hash-password

.. code-block:: text
    :class: ignore
    :emphasize-lines: 11

    Symfony Password Hash Utility
    =============================

     Type in your password to be hashed:
     >

     ------------------ ---------------------------------------------------------------------------------------------------
      Key                Value
     ------------------ ---------------------------------------------------------------------------------------------------
      Hasher used        Symfony\Component\PasswordHasher\Hasher\MigratingPasswordHasher
      Password hash      $argon2id$v=19$m=65536,t=4,p=1$BQG+jovPcunctc30xG5PxQ$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s
     ------------------ ---------------------------------------------------------------------------------------------------

     ! [NOTE] Self-salting hasher used: the hasher generated its own built-in salt.


     [OK] Password hashing succeeded

إنشاء مدير
-------------------

.. index::
    single: Command;dbal:run-sql

أدخل المستخدم المدير عبر عبارة SQL:

.. code-block:: terminal

    $ symfony console dbal:run-sql "INSERT INTO admin (id, username, roles, password) \
      VALUES (nextval('admin_id_seq'), 'admin', '[\"ROLE_ADMIN\"]', \
      '\$argon2id\$v=19\$m=65536,t=4,p=1\$BQG+jovPcunctc30xG5PxQ\$TiGbx451NKdo+g9vLtfkMy4KjASKSOcnNxjij4gTX1s')"

لاحظ التخليص escaping من علامة " $ '' في عمود كلمة المرور ؛ قم بهذا (escaping) لكل منهم !

إعداد مصادقة الأمان Security Authentication
------------------------------------------------------------

.. index::
    single: Command;make:security:form-login
    single: Security;Authenticator
    single: Security;Form Login
    single: Login
    single: Logout

الآن بعد أن أصبح لدينا مستخدم مدير ، يمكننا تأمين الواجهة الخلفية للمدير. يدعم Symfony العديد من استراتيجيات المصادقة. دعونا نستخدم نظام مصادقة نموذجي وشائع.

قم بتشغيل الأمر " make:security:form-login '' لتحديث إعداد الأمان (The security configuration ) ، وإنشاء قالب تسجيل دخول ، وإنشاء *authenticator*:

.. code-block:: terminal
    :class: answers(SecurityController||yes)

    $ symfony console make:security:form-login

سمِّ وحدة التحكم Controller  " SecurityController '' ، وقم بإنشاء عنوان URL " / logout '' (" نعم '').

قام الأمر بتحديث إعداد الأمان The security configuration  لتوصيل الفئات classes  التي تم إنشاؤها:

.. code-block:: diff
    :class: ignore
    :emphasize-lines: 9

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -15,7 +15,15 @@ security:
                 security: false
             main:
                 lazy: true
    -            provider: users_in_memory
    +            provider: app_user_provider
    +            form_login:
    +                login_path: app_login
    +                check_path: app_login
    +                enable_csrf: true
    +            logout:
    +                path: app_logout
    +                # where to redirect after logout
    +                # target: app_any_route

                 # activate different ways to authenticate
                 # https://symfony.com/doc/current/security.html#the-firewall

.. index::
    single: Command;debug:router
    single: Routing;Debug
    single: Debug;Routing

.. tip::

    كيف أتذكر أن مسار EasyAdmin هو ``admin`` (كما هو محدد في ``App\Controller\Admin\DashboardController``)؟ لا أتذكر. يمكنك النظر للملف وأيضا إذا قمت بتشغيل الأمر التالي الذي يوضح الارتباط بين أسماء المسارات route names  والمسارات paths :

    .. code-block:: terminal

        $ symfony console debug:router

إضافة قواعد التحكم في الوصول إلى التفويض Authorization Access Control Rules
-------------------------------------------------------------------------------------------------------------

.. index::
    single: Security;Authorization
    single: Security;Access Control

يتكون نظام الأمان من جزأين: *المُصادقة* و *التفويض*. عند إنشاء مستخدم المدير، أعطينا لهم دور ``ROLE_ADMIN``. لنقصر قسم ``/admin`` على المستخدمين الذين لديهم هذا الدور عن طريق إضافة قاعدة إلى ``access_control``:

.. code-block:: diff
    :emphasize-lines: 8

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -34,7 +34,7 @@ security:
         # Easy way to control access for large sections of your site
         # Note: Only the *first* access control that matches will be used
         access_control:
    -        # - { path: ^/admin, roles: ROLE_ADMIN }
    +        - { path: ^/admin, roles: ROLE_ADMIN }
             # - { path: ^/profile, roles: ROLE_USER }

     when@test:

تُقيد قواعد ``access_control`` إمكانية الوصول عن طريق التعابير النمطية (regular expressions). عند محاولة الاتصال عن طريق رابط يبدأ بـ ``/admin``، سوف يقوم النظام الأمني بالتحقق من دور الـ ``ROLE_ADMIN`` علي المستخدم الذي قام بتسجيل الدخول.

المُصادقة عبر نموذج تسجيل الدخول
------------------------------------------------------------

إذا حاولت الوصول الي خلفية الادارة (admin backend)، يجب ان يتم تحويلك الآن الي صفحة تسجيل الدخول وسيتم مطالبتك بإدخال بينات تسجيل الدخول وكلمة مرور:

.. figure:: screenshots/easy-admin-login.png
    :alt: /login/
    :align: center
    :figclass: with-browser

قم بتسجيل الدخول بإستخدام ``admin`` واي كلمة مرور اخترتها مُسبقاً. لو قمت بنسخ امر الـ SQL الخاص بي كما هو فكلمة المرور هي ``admin``.

لاحظ أن EasyAdmin تُدرك نظام تصادق سيمفوني بشكل تلقائي:

.. figure:: screenshots/easy-admin-secured.png
    :alt: /admin/
    :align: center
    :figclass: with-browser

حاول أن تضغط علي رابط تسجيل الخروج "Sign out". إنه بحوزتك! خلفية مسؤول (backend admin) مؤمنة بالكامل.

.. index::
    single: Command;make:registration-form

.. note::

    إذا كنت تريد إنشاء نموذج مُصادقة مكتمل المميزات، لُتلقي نظرة علي أمر ``make:registration-form``.

.. sidebar:: الذهاب أبعد من ذلك

    * `مراجع أمان سيمفوني`_؛

    * `SymfonyCasts Security tutorial`_؛

    * `كيف تنشئ نموذج تسجيل دخول`_ في تطبيقات سيمفوني؛

    * `ورقة الغش لأمان سيمفوني`_.

.. _`مراجع أمان سيمفوني`: https://symfony.com/doc/current/security.html
.. _`SymfonyCasts Security tutorial`: https://symfonycasts.com/screencast/symfony-security
.. _`كيف تنشئ نموذج تسجيل دخول`: https://symfony.com/doc/current/security/form_login_setup.html
.. _`ورقة الغش لأمان سيمفوني`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/security_en_44.pdf
