توطين تطبيق
=====================

مع وجود جمهور دولي ، تمكنت سيمفوني من التعامل مع التدويل (i18n) والتعريب (l10n) خارج الصندوق منذ أي وقت مضى. لا يعني ترجمة التطبيق إلى ترجمة الواجهة فحسب ، بل يتعلق أيضًا بصيغ الجمع وتنسيق التاريخ والعملة وعناوين URL والمزيد.

تدويل عناوين المواقع
--------------------------------------

.. index::
    single: Components;Routing
    single: Routing;Locale
    single: Routing;Requirements
    single: Attributes;Route

الخطوة الأولى لتدويل موقع الويب هي تدويل عناوين URL. عند ترجمة واجهة موقع ويب ، يجب أن يكون عنوان URL مختلفًا لكل لغة بحيث يكون لطيفًا مع ذاكرات HTTP المؤقتة (لا تستخدم أبدًا عنوان URL نفسه وتخزين الإعدادات المحلية في الجلسة).

استخدم معلمة المسار `` _locale`` الخاصة للإشارة إلى الإعدادات المحلية في المسارات:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/', name: 'homepage')]
    +    #[Route('/{_locale}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

في الصفحة الرئيسية ، يتم الآن تعيين الإعدادات الداخلية داخليًا وفقًا لعنوان URL ؛ على سبيل المثال ، إذا قمت بالضغط على ``/fr/``، ``$request->getLocale()`` إرجاع ``fr``.

نظرًا لأنك لن تتمكن على الأرجح من ترجمة المحتوى في جميع الأماكن الصحيحة ، فاحصر على ما تريد دعمه:

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 8

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/{_locale}/', name: 'homepage')]
    +    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

يمكن تقييد كل معلمة توجيه بتعبير منتظم داخل `` <`` ``> `` `. يطابق المسار "الصفحة الرئيسية" الآن فقط عندما تكون المعلمة `` _locale` `` en`` أو `fr``. حاول ضرب `` / es / `` ، يجب أن يكون لديك 404 حيث لا توجد طرق مطابقة.

نظرًا لأننا سنستخدم نفس المتطلبات في جميع المسارات تقريبًا ، فلننقلها إلى معلمة حاوية:

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -9,5 +9,6 @@ parameters:
         admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"
         default_base_url: 'http://127.0.0.1'
    +    app.supported_locales: 'en|fr'

     services:
         # default configuration for services in *this* file
    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -28,7 +28,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/{_locale<en|fr>}/', name: 'homepage')]
    +    #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/index.html.twig', [

يمكن إضافة لغة عن طريق تحديث المعلمة `` app.supported_languages``.

أضف بادئة مسار الإعدادات المحلية نفسها إلى عناوين URL الأخرى:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -38,7 +38,7 @@ final class ConferenceController extends AbstractController
         }

         #[Cache(smaxage: 3600)]
    -    #[Route('/conference_header', name: 'conference_header')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {
             return $this->render('conference/header.html.twig', [
    @@ -46,8 +46,8 @@ final class ConferenceController extends AbstractController
             ]);
         }

         #[RateLimit('comment_submission', methods: ['POST'])]
    -    #[Route('/conference/{slug:conference}', name: 'conference')]
    +    #[Route('/{_locale<%app.supported_locales%>}/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,
             Conference $conference,

نحن على وشك الإنتهاء. لم يعد لدينا طريق يطابق ``/`` بعد الآن. دعنا نضيفه ونجعله يعيد التوجيه إلى ``/en/``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -27,6 +27,12 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Route('/')]
    +    public function indexNoLocale(): Response
    +    {
    +        return $this->redirectToRoute('homepage', ['_locale' => 'en']);
    +    }
    +
         #[Cache(smaxage: 3600)]
         #[Route('/{_locale<%app.supported_locales%>}/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response

الآن وبعد أن أصبحت جميع المسارات الرئيسية على دراية بالإعدادات المحلية ، لاحظ أن عناوين URL التي تم إنشاؤها على الصفحات تأخذ الإعدادات المحلية الحالية في الاعتبار تلقائيًا.

اضافة عامل تحويل المحليات
-----------------------------------------------

.. index::
    single: Twig;path
    single: Twig;Locale

للسماح للمستخدمين بالتبديل من الإعدادات المحلية  الافتراضية ``en`` إلى أخرى ، دعنا نضيف محوّل في الرأس:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -34,6 +34,16 @@
                                         Admin
                                     </a>
                                 </li>
    +<li class="nav-item dropdown">
    +    <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
    +        data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    +        English
    +    </a>
    +    <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
    +        <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>
    +        <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'fr'}) }}">Français</a></li>
    +    </ul>
    +</li>
                             </ul>
                         </div>
                     </div>

للتبديل إلى لغة أخرى ، نقوم صراحةً بتمرير معلمة المسار `` _locale`` إلى دالة ``path()``.

.. index::
    single: Twig;app.request
    single: Twig;locale_name

قم بتحديث القالب لعرض اسم الإعدادات المحلية الحالي بدلاً من "الإنجليزية" المرمزة:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        English
    +        {{ app.request.locale|locale_name(app.request.locale) }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

``app`` هو متغير  Twig عالمي يتيح الوصول إلى الطلب الحالي. لتحويل الإعدادات المحلية إلى سلسلة بشرية قابلة للقراءة ، نحن نستخدم ``locale_name`` مصفاة Twig.

.. index::
    single: Components;String

اعتمادًا على الإعدادات المحلية ، لا يتم دائمًا كتابة اسم الإعدادات المحلية. لتكبير حجم الجمل بشكل صحيح ، نحتاج إلى عامل تصفية يدرك Unicode ، كما هو منصوص عليه في المكون Symfony String وتطبيق Twig الخاص به:

.. code-block:: terminal

    $ symfony composer req twig/string-extra

.. index::
    single: Twig;u.title

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -37,7 +37,7 @@
     <li class="nav-item dropdown">
         <a class="nav-link dropdown-toggle" href="#" id="dropdown-language" role="button"
             data-bs-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
    -        {{ app.request.locale|locale_name(app.request.locale) }}
    +        {{ app.request.locale|locale_name(app.request.locale)|u.title }}
         </a>
         <ul class="dropdown-menu dropdown-menu-right" aria-labelledby="dropdown-language">
             <li><a class="dropdown-item" href="{{ path('homepage', {_locale: 'en'}) }}">English</a></li>

يمكنك الآن التبديل من الفرنسية إلى الإنجليزية عبر المحول وتتكيف الواجهة بأكملها بشكل جيد للغاية:

.. figure:: screenshots/intl-switcher.png
    :alt: /fr/conference/amsterdam-2019
    :align: center
    :figclass: with-browser

ترجمة الواجهة
-------------------------

.. index::
    single: Components;Translation
    single: Translation
    single: Twig;trans

قد تكون ترجمة كل جملة على موقع ويب كبير أمرًا شاقًا ، لكن لحسن الحظ ، لدينا فقط عدد قليل من الرسائل على موقعنا. لنبدأ بكل الجمل في الصفحة الرئيسية:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -20,7 +20,7 @@
                 <nav class="navbar navbar-expand-xl navbar-light bg-light">
                     <div class="container mt-4 mb-3">
                         <a class="navbar-brand me-4 pr-2" href="{{ path('homepage') }}">
    -                        &#128217; Conference Guestbook
    +                        &#128217; {{ 'Conference Guestbook'|trans }}
                         </a>

                         <button class="navbar-toggler border-0" type="button" data-bs-toggle="collapse" data-bs-target="#header-menu" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Show/Hide navigation">
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -4,7 +4,7 @@

     {% block body %}
         <h2 class="mb-5">
    -        Give your feedback!
    +        {{ 'Give your feedback!'|trans }}
         </h2>

         {% for row in conferences|batch(4) %}
    @@ -21,7 +21,7 @@

                                 <a href="{{ path('conference', { slug: conference.slug }) }}"
                                    class="btn btn-sm btn-primary stretched-link">
    -                                View
    +                                {{ 'View'|trans }}
                                 </a>
                             </div>
                         </div>

يبحث عامل تصفية Twig عن طريق ترجمة المدخلات المعطاة إلى الإعدادات المحلية الحالية. إذا لم يتم العثور عليه ، فإنه يعود إلى * الإعدادات المحلية الافتراضية * كما تم تهيئته في "التكوين / الحزم / translation.yaml``:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 2

    framework:
        default_locale: en
        translator:
            default_path: '%kernel.project_dir%/translations'
            fallbacks:
                - en

لاحظ أن ترجمة شريط أدوات تصحيح الويب  "tab" قد تحولت إلى اللون الأحمر:

.. figure:: screenshots/intl-wdt.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

يخبرنا أن 3 رسائل لم تترجم بعد.

انقر فوق "علامة التبويب" لسرد جميع الرسائل التي لم تجد Symfony ترجمة لها:

.. figure:: screenshots/intl-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

توفير الترجمات
---------------------------

كما كنت قد رأيت في  ``config/packages/translation.yaml`` ، يتم تخزين الترجمات ضمن دليل الجذر للترجمات ، والذي تم إنشاؤه تلقائيًا لنا.

بدلاً من إنشاء ملفات الترجمة يدويًا ، استخدم الأمر ``translation:extract``:

.. code-block:: terminal

    $ symfony console translation:extract fr --force --domain=messages

يقوم هذا الأمر بإنشاء ملف ترجمة (علامة `` ``--force`) للإعدادات المحلية` `fr`` ونطاق` messages` (الذي يحتوي على جميع الرسائل غير الأساسية مثل أخطاء التحقق من الصحة أو الأمان).

قم بتحرير ملف ``translations/messages+intl-icu.fr.xlf`` وترجمة الرسائل باللغة الفرنسية. لا تتكلم الفرنسية دعني اساعدك:

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/translations/messages+intl-icu.fr.xlf
    +++ w/translations/messages+intl-icu.fr.xlf
    @@ -7,15 +7,15 @@
         <body>
           <trans-unit id="eOy4.6V" resname="Conference Guestbook">
             <source>Conference Guestbook</source>
    -        <target>__Conference Guestbook</target>
    +        <target>Livre d'Or pour Conferences</target>
           </trans-unit>
           <trans-unit id="LNAVleg" resname="Give your feedback!">
             <source>Give your feedback!</source>
    -        <target>__Give your feedback!</target>
    +        <target>Donnez votre avis !</target>
           </trans-unit>
           <trans-unit id="3Mg5pAF" resname="View">
             <source>View</source>
    -        <target>__View</target>
    +        <target>Sélectionner</target>
           </trans-unit>
         </body>
       </file>

.. code-block:: xml
    :caption: translations/messages+intl-icu.fr.xlf
    :class: hide

    <?xml version="1.0" encoding="utf-8"?>
    <xliff xmlns="urn:oasis:names:tc:xliff:document:1.2" version="1.2">
    <file source-language="en" target-language="fr" datatype="plaintext" original="file.ext">
        <header>
        <tool tool-id="symfony" tool-name="Symfony" />
        </header>
        <body>
        <trans-unit id="LNAVleg" resname="Give your feedback!">
            <source>Give your feedback!</source>
            <target>Donnez votre avis !</target>
        </trans-unit>
        <trans-unit id="3Mg5pAF" resname="View">
            <source>View</source>
            <target>Sélectionner</target>
        </trans-unit>
        <trans-unit id="eOy4.6V" resname="Conference Guestbook">
            <source>Conference Guestbook</source>
            <target>Livre d'Or pour Conferences</target>
        </trans-unit>
        </body>
    </file>
    </xliff>

لاحظ أننا لن نترجم جميع القوالب ، لكن لا تتردد في القيام بذلك:

.. figure:: screenshots/intl-translated.png
    :alt: /fr/
    :align: center
    :figclass: with-browser

ترجمة النماذج
-------------------------

.. index::
    single: Translation;Form
    single: Form;Translation

يتم عرض ملصقات النماذج تلقائيًا بواسطة سيمفوني عبر نظام الترجمة. انتقل إلى صفحة مؤتمر وانقر على علامة التبويب "الترجمة" من شريط أدوات تصحيح الويب ؛ يجب أن ترى جميع التصنيفات جاهزة للترجمة:

.. figure:: screenshots/intl-form-profiler.png
    :alt: /_profiler/64282d?panel=translation
    :align: center
    :figclass: with-browser

توطين التواريخ
---------------------------

.. index::
    single: Localization
    single: Twig;format_datetime
    single: Twig;format_time
    single: Twig;format_date
    single: Twig;format_currency
    single: Twig;format_number

إذا قمت بالتبديل إلى الفرنسية وانتقلت إلى صفحة ويب خاصة بالمؤتمر تحتوي على بعض التعليقات ، فستلاحظ أن تواريخ التعليقات يتم تحديدها تلقائيًا. يعمل هذا لأننا استخدمنا عامل تصفية twig وهو ``format_datetime``، والذي يكون على دراية بالإعدادات المحلية (``{{comment. createdAt | format_datetime ('medium' ، 'short')}}``).

تعمل الترجمة في التواريخ والأوقات (`` format_time``) والعملات (`` format_currency``) والعملات (`` format_currency``) والأرقام (`` format_number``) بشكل عام (النسب المئوية والمدة والتهجئة) خارج ، ...).

ترجمة التعددية
---------------------------

.. index::
    single: Translation;Plurals
    single: Translation;Conditions

تعد إدارة الجمع في الترجمات أحد استخدامات المشكلة العامة المتمثلة في اختيار ترجمة بناءً على شرط.

في صفحة المؤتمر ، نعرض عدد التعليقات: ``There are 2 comments``. للتعليق ، نعرض ``There are 1 comments`` ، وهذا خطأ. تعديل القالب لتحويل الجملة إلى رسالة قابلة للترجمة:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -44,7 +44,7 @@
                             </div>
                         </div>
                     {% endfor %}
    -                <div>There are {{ comments|length }} comments.</div>
    +                <div>{{ 'nb_of_comments'|trans({count: comments|length}) }}</div>
                     {% if previous >= 0 %}
                         <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
                     {% endif %}

لهذه الرسالة ، استخدمنا استراتيجية أخرى للترجمة. بدلاً من الاحتفاظ بالإصدار باللغة الإنجليزية في القالب ، استبدلناه بمعرف فريد. تعمل هذه الإستراتيجية بشكل أفضل مع كمية معقدة من النص.

قم بتحديث ملف الترجمة عن طريق إضافة الرسالة الجديدة:

.. code-block:: diff
    :caption: patch_file

    --- i/translations/messages+intl-icu.fr.xlf
    +++ w/translations/messages+intl-icu.fr.xlf
    @@ -17,6 +17,10 @@
             <source>Conference Guestbook</source>
             <target>Livre d'Or pour Conferences</target>
         </trans-unit>
    +    <trans-unit id="Dg2dPd6" resname="nb_of_comments">
    +        <source>nb_of_comments</source>
    +        <target>{count, plural, =0 {Aucun commentaire.} =1 {1 commentaire.} other {# commentaires.}}</target>
    +    </trans-unit>
         </body>
     </file>
     </xliff>

لم ننته بعد لأننا نحتاج الآن إلى توفير الترجمة الإنجليزية. قم بإنشاء ``translations/messages+intl-icu.en.xlf``:

.. code-block:: xml
    :caption: translations/messages+intl-icu.en.xlf
    :emphasize-lines: 10

    <?xml version="1.0" encoding="utf-8"?>
    <xliff xmlns="urn:oasis:names:tc:xliff:document:1.2" version="1.2">
      <file source-language="en" target-language="en" datatype="plaintext" original="file.ext">
        <header>
          <tool tool-id="symfony" tool-name="Symfony" />
        </header>
        <body>
          <trans-unit id="maMQz7W" resname="nb_of_comments">
            <source>nb_of_comments</source>
            <target>{count, plural, =0 {There are no comments.} one {There is one comment.} other {There are # comments.}}</target>
          </trans-unit>
        </body>
      </file>
    </xliff>

تحديث الاختبارات الوظيفية
------------------------------------------------

لاتنسى تعديل الاختبارات التوظيفية لأخذ الروابط ومحتوى التعديلات:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -16,7 +16,7 @@ class ConferenceControllerTest extends WebTestCase
         public function testIndex(): void
         {
             $client = static::createClient();
    -        $client->request('GET', '/');
    +        $client->request('GET', '/en/');

             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
    @@ -29,7 +29,7 @@ class ConferenceControllerTest extends WebTestCase
             $berlin = ConferenceFactory::createOne(['city' => 'Berlin', 'year' => '2021', 'isInternational' => false]);
             CommentFactory::createOne(['conference' => $berlin]);

    -        $client->request('GET', '/conference/berlin-2021');
    +        $client->request('GET', '/en/conference/berlin-2021');
             $client->submitForm('Submit', [
                 'comment[author]' => 'Fabien',
                 'comment[text]' => 'Some feedback from an automated functional test',
    @@ -50,7 +50,7 @@ class ConferenceControllerTest extends WebTestCase
             ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
             CommentFactory::createOne(['conference' => $amsterdam]);

    -        $crawler = $client->request('GET', '/');
    +        $crawler = $client->request('GET', '/en/');

             $this->assertCount(2, $crawler->filter('h4'));

    @@ -59,6 +59,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertPageTitleContains('Amsterdam');
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    -        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +        $this->assertSelectorExists('div:contains("There is one comment")');
         }
     }

.. sidebar:: الذهاب أبعد من ذلك

    * `Translating Messages using the ICU formatter`_؛

    * `Using Twig translation filters`_.

.. _`Translating Messages using the ICU formatter`: https://symfony.com/doc/current/translation/message_format.html
.. _`Using Twig translation filters`: https://symfony.com/doc/current/translation/templates.html#translation-filters
