التخزين المؤقت للأداء
========================================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

مشاكل الأداء قد تأتي مع الشعبية. بعض الأمثلة النموذجية: فهارس قاعدة البيانات المفقودة أو طن من طلبات SQL لكل صفحة.لن تواجه أي مشكلة في قاعدة بيانات فارغة ، ولكن مع زيادة عدد الزيارات والبيانات المتنامية ، قد تظهر في مرحلة ما.

اخفاء معلومات خاصة بالHTTP
---------------------------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

يعد استخدام استراتيجيات التخزين المؤقت لـ HTTP طريقة رائعة لزيادة أداء المستخدمين النهائيين بأقل جهد ممكن. أضف ذاكرة التخزين المؤقت للوكيل العكسي في الإنتاج لتمكين التخزين المؤقت ، واستخدم `CDN`_ للتخزين المؤقت على الحافة للحصول على أداء أفضل.

دعنا نخفي الصفحة الرئيسية لمدة ساعة

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -14,6 +14,7 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\Cache;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    @@ -27,6 +28,7 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {

تُهيّئ السمة ``#[Cache]`` انتهاء صلاحية ذاكرة التخزين المؤقت للوكلاء العكسيين عبر وسيطها ``smaxage``؛ استخدم ``maxage`` للتحكم في ذاكرة التخزين المؤقت للمتصفح. يتم التعبير عن الوقت بالثواني (ساعة واحدة = 60 دقيقة = 3600 ثانية). ومثل التوجيه (routing) أو تحديد المعدل (rate limiting)، تُعلَن سياسة التخزين المؤقت في المكان الذي تنطبق فيه تمامًا: على وحدة التحكم (controller).

يعد التخزين المؤقت لصفحة المؤتمر أكثر صعوبة لأنه أكثر ديناميكية. يمكن لأي شخص إضافة تعليق في أي وقت ، ولا يريد أحد الانتظار لمدة ساعة لرؤيته عبر الإنترنت. في مثل هذه الحالات ، استخدم استراتيجية * التحقق من صحة HTTP.

تنشيط Symfony HTTP Cache Kernel
------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

لاختبار استراتيجية ذاكرة التخزين المؤقت HTTP ، فعّل الوكيل العكسي Symfony HTTP، ولكن فقط في بيئة "التطوير" (لبيئة "الإنتاج"، سنستخدم حلاً "أكثر متانة"):

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

إلى جانب كونه وكيل HTTP كامل العكسي ، يضيف الوكيل العكسي Symfony HTTP (عبر فئة ``HttpCache``) بعض معلومات التصحيح الجيدة كرؤوس HTTP. وهذا يساعد بشكل كبير في التحقق من صحة رؤوس ذاكرة التخزين المؤقت التي وضعناها.

التحقق من ذلك على الصفحة الرئيسية:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

بالنسبة للطلب الأول ، يخبرك خادم التخزين المؤقت بأنه كان ``miss`` وأنه أجرى ``store`` لتخزين الاستجابة مؤقتًا. تحقق من رأس ``cache-control`` لرؤية استراتيجية ذاكرة التخزين المؤقت المكونة.

للطلبات اللاحقة ، يتم تخزين الاستجابة مؤقتًا (تم أيضًا تحديث ``age``):

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

تجنب طلبات SQL باستخدام ESI
--------------------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

يقوم مستمع `TwigEventListener` بإدخال متغير عمومي في Twig مع جميع كائنات المؤتمر. يفعل ذلك لكل صفحة واحدة من الموقع. ربما يكون هدفًا رائعًا للتحسين.

لن تضيف مؤتمرات جديدة كل يوم ، وبالتالي فإن الكود يبحث عن نفس البيانات بالضبط من قاعدة البيانات مرارًا وتكرارًا.

قد نرغب في تخزين ذاكرة التخزين المؤقت لأسماء المؤتمرات والبزاقات باستخدام ذاكرة التخزين المؤقت Symfony ، ولكن كلما أمكن ذلك ، أود الاعتماد على بنية تخزين HTTP المؤقتة.

عندما تريد التخزين المؤقت لجزء من الصفحة ، انقله خارج طلب HTTP الحالي عن طريق إنشاء *طلب فرعي*. *ESI* هي مباراة مثالية لحالة الاستخدام هذه. ESI هي وسيلة لتضمين نتيجة طلب HTTP في طلب آخر.

إنشاء وحدة تحكم تقوم بإرجاع جزء HTML الذي يعرض المؤتمرات فقط:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,14 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(

قم بإنشاء القالب المقابل:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

اضغط على ``/conference_header`` للتحقق من أن كل شيء يعمل بشكل جيد.

.. index::
    single: Twig;render
    single: Twig;path

حان الوقت للكشف عن الخدعة! قم بتحديث تخطيط Twig للاتصال بوحدة التحكم التي أنشأناها للتو:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

و *voilà*. قم بتحديث الصفحة ولا يزال موقع الويب يعرض نفسه.

.. tip::

    استخدم لوحة ملفات التعريف "طلب / استجابة" Symfony لمعرفة المزيد عن الطلب الرئيسي وطلباته الفرعية.

الآن ، في كل مرة تضغط فيها صفحة في المتصفح ، يتم تنفيذ طلبين HTTP ، واحد للرأس وواحد للصفحة الرئيسية. لقد جعلت الأداء أسوأ. تهانينا!

يتم إجراء مكالمة HTTP لرأس المؤتمر حاليًا داخليًا بواسطة Symfony ، لذلك لا يتم تضمين رحلة ذهاب HTTP. هذا يعني أيضًا أنه لا توجد وسيلة للاستفادة من رؤوس ذاكرة التخزين المؤقت HTTP.

تحويل المكالمة إلى HTTP "حقيقي" باستخدام ESI.

أولاً ، قم بتمكين دعم ESI:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

ثم ، استخدم ``render_esi``  بدلا من ``render``:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

إذا اكتشفت Symfony وكيلًا عكسيًا يعرف كيفية التعامل مع ESIs ، فسيمكنه الدعم تلقائيًا (إن لم يكن كذلك ، فإنه يعود لتقديم الطلب الفرعي بشكل متزامن).

بما أن الوكيل العكسي لSymfony يدعم ESIs ، فلنراجع سجلاته (قم بإزالة ذاكرة التخزين المؤقت أولاً - انظر "Purging" أدناه):

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

قم بالتحديث عدة مرات: استجابة `` / `` مخزنة مؤقتًا والاستجابة `` Conference_header/ `` ليست كذلك. لقد حققنا شيئًا رائعًا: وجود الصفحة الكاملة في ذاكرة التخزين المؤقت ولكن لا يزال هناك جزء ديناميكي واحد.

هذا ليس ما نريده بالرغم من ذلك. قم بتخزين صفحة الرأس في ذاكرة التخزين المؤقت لمدة ساعة ، بغض النظر عن أي شيء آخر:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {

تم تمكين ذاكرة التخزين المؤقت الآن لكلا من الطلبين:

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

يحتوي رأس ``x-symfony-cache`` على عنصرين: الطلب الرئيسي ``/`` وطلب فرعي (``conference_header ``ESI). كلاهما في ذاكرة التخزين المؤقت (``الطازجة``).

يمكن أن تختلف استراتيجية التخزين المؤقت عن الصفحة الرئيسية و ESIs الخاصة بها. إذا كانت لدينا صفحة "about" ، فقد نرغب في تخزينها لمدة أسبوع في ذاكرة التخزين المؤقت ، ولا يزال يتم تحديث الرأس كل ساعة.

قم بإزالة المستمع لأننا لسنا بحاجة إليه بعد الآن:

.. code-block:: terminal

    $ rm src/EventListener/TwigEventListener.php

تطهير ذاكرة التخزين المؤقت HTTP للاختبار
-----------------------------------------------------------------------

يصبح اختبار موقع الويب في متصفح أو عبر الاختبارات الآلية أكثر صعوبة مع طبقة التخزين المؤقت.

يمكنك إزالة كل ذاكرة التخزين المؤقت HTTP يدويًا بحذف دليل ``var/cache/dev/http_cache/``:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

لا تعمل هذه الإستراتيجية بشكل جيد إذا كنت تريد فقط إبطال بعض عناوين URL أو إذا كنت تريد دمج إلغاء صلاحية ذاكرة التخزين المؤقت في اختباراتك الوظيفية. دعنا نضيف نقطة نهاية HTTP صغيرة ، للمسؤول فقط لإبطال بعض عناوين URL:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

تم تقييد وحدة التحكم الجديدة على طريقة HTTP`` PURGE``. هذه الطريقة ليست في معيار HTTP ، لكنها تستخدم على نطاق واسع لإبطال ذاكرة التخزين المؤقت.

بشكل افتراضي ، لا يمكن أن تحتوي معلمات المسار على ``/`` لأنها تفصل بين أجزاء عنوان URL. يمكنك تجاوز هذا القيد لمعلمة المسار الأخيرة ، مثل ``uri`` ، عن طريق تعيين نمط المتطلبات الخاص بك (``. *``).

يمكن أن تبدو الطريقة التي نحصل بها على مثيل `` HttpCache `` غريبة بعض الشيء ؛ نحن نستخدم فئة مجهولة حيث الوصول إلى الفصل "الحقيقي" غير ممكن. يلف مثيل `` HttpCache `` النواة الحقيقية ، التي لا تدرك طبقة التخزين المؤقت كما ينبغي.

قم بإبطال الصفحة الرئيسية ورأس المؤتمر عبر مكالمات cURL التالية:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

يعرض الأمر الفرعي `` symfony var: export SYMFONY_PROJECT_DEFAULT_ROUTE_URL `` عنوان URL الحالي لخادم الويب المحلي.

.. note::

    لا تحتوي وحدة التحكم على اسم مسار حيث لن تتم الإشارة إليه مطلقًا في الكود.

تعطيل ذاكرة التخزين المؤقت HTTP في التطوير
----------------------------------------------------------

كانت ذاكرة التخزين المؤقت HTTP رائعة للتحقق من رؤوس التخزين المؤقت ولتعلّم كيفية تطهير الإدخالات القديمة. لكن تفعيل وكيل عكسي في بيئة التطوير أمر غير معتاد، وسرعان ما يصبح عائقًا: تُقدَّم الاستجابات من ذاكرة التخزين المؤقت بينما تعمل على الكود، بل تُقدَّم بعض أصول vendor باستجابة فارغة بسبب قيد قديم في HttpCache مع استجابات الملفات.

الآن بعد أن تم التحقق من كل شيء، قم بتعطيلها؛ سيتولى Varnish الأمر في الإنتاج:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -14,7 +14,3 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    -
    -when@dev:
    -    framework:
    -        http_cache: true

تجميع مسارات مماثلة بإستعمل Prefix
----------------------------------------------------------

.. index::
    single: Attributes;Route

المساران في وحدة تحكم المشرف لهما نفس البادئة `` admin/ `` بدلاً من تكرارها في جميع المسارات ، قم بإعادة بناء الطرق لتكوين البادئة على الفصل نفسه:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

التخزين المؤقت للعمليات المكثفة للذاكرة / وحدة المعالجة المركزية
-----------------------------------------------------------------------------------------------------------------------

.. index::
    single: Process
    single: Components;Process

ليس لدينا خوارزميات تستهلك الكثير من الذاكرة أو وحدة المعالجة المركزية على الموقع. للتحدث عن * ذاكرة التخزين المؤقت المحلية / local caches * ، دعنا ننشئ أمرًا يعرض الخطوة الحالية التي نعمل عليها (لنكون أكثر دقة ، اسم علامة Git المرتبط بتنفيذ Git الحالي).

يسمح لك مكون  Symfony Process بتشغيل أمر والحصول على النتيجة مرة أخرى (standard and error output).

بناء الأمر:

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand
    {
        public function __invoke(OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Cache
    single: Components;Cache

ماذا لو أردنا تخزين النتيجة  لبضع دقائق؟ استخدم ذاكرة التخزين المؤقت Symfony.

يحقن Symfony الخدمات المُلمَّحة بنوعها (type-hinted) في طريقة ``__invoke()`` للأمر، بنفس الطريقة التي يفعلها مع وسائط وحدة التحكم. لُف الكود بمنطق ذاكرة التخزين المؤقت:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -6,15 +6,21 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand
     {
    -    public function __invoke(OutputInterface $output): int
    +    public function __invoke(OutputInterface $output, CacheInterface $cache): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

يتم استدعاء العملية الآن فقط إذا لم يكن عنصر `` app.current_step `` في ذاكرة التخزين المؤقت.

التنميط ومقارنة الأداء
------------------------------------------

لا تقم بإضافة ذاكرة التخزين المؤقت بدون مبالاة. ضع في اعتبارك أن إضافة بعض ذاكرة التخزين المؤقت يضيف طبقة من التعقيد. وبما أننا جميعًا سيئون جدًا في تخمين ما سيكون سريعًا وما هو بطيء ، فقد ينتهي بك الأمر في موقف حيث تجعل ذاكرة التخزين المؤقت تطبيقك أبطأ.

قم دائمًا بقياس تأثير إضافة ذاكرة التخزين المؤقت باستخدام أداة للتحليل مثل `Blackfire`_.

ارجع إلى خطوة "الأداء" لمعرفة المزيد حول كيفية استخدام Blackfire لاختبار الكود قبل النشر.

إعداد ذاكرة التخزين المؤقت للخادم الوكيل العكسي Reverse Proxy Cache  للإنتاج
----------------------------------------------------------------------------------------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

بدلاً من استخدام وكيل Symfony العكسي في الإنتاج، سنستخدم وكيل Varnish العكسي "الأكثر متانة".

أضف Varnish إلى خدمات Upsun:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -6,6 +6,15 @@ services:
         database:
             type: postgresql:16

    +    varnish:
    +        type: varnish:9.0
    +        relationships:
    +            application: 'app:http'
    +        configuration:
    +            vcl: !include
    +                type: string
    +                path: config.vcl
    +
     applications:

.. index::
    single: Upsun;Routes

استخدم Varnish كنقطة دخول رئيسية في المسارات:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

أخيرًا ، قم بإنشاء ملف `` config.vcl '' لإعداد Varnish:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

تمكين دعم ESI على Varnish
------------------------------------

يجب تمكين دعم ESI على Varnish بشكل صريح لكل طلب. لجعله عالميًا، يستخدم Symfony الرؤوس القياسية `` Surrogate-Capability `` و `` Surrogate-Control `` للتفاوض بشأن دعم ESI:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

تطهير Varnish Cache
------------------------

ربما لا تكون هناك حاجة أبدًا لإبطال ذاكرة التخزين المؤقت في الإنتاج ، باستثناء الأغراض الطارئة وربما في الفروع غير ``master``. إذا كنت بحاجة إلى مسح ذاكرة التخزين المؤقت في كثير من الأحيان ، فربما يعني هذا أنه يجب تعديل استراتيجية التخزين المؤقت (عن طريق خفض TTL أو باستخدام استراتيجية التحقق من الصحة بدلاً من انتهاء الصلاحية).

على أي حال ، دعنا نرى كيفية إعداد Varnish لإبطال ذاكرة التخزين المؤقت:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

في الحياة الواقعية ، ربما تقيد عناوين IP بدلاً من ذلك كما هو موضح في `مستندات Varnish`_.

امسح بعض عناوين URL:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

تبدو عناوين URL غريبة بعض الشيء لأن عناوين URL التي يتم إرجاعها بواسطة ``env:url`` تنتهي بالفعل بـ ``/``.

.. sidebar:: الذهاب أبعد من ذلك

    * `Cloudflare`_ ، منصة السحابة العالمية؛

    * `Varnish HTTP Cache docs`_؛

    * `مواصفات ESI`_ و `موارد مطوري ESI`_؛

    * `نموذج التحقق من ذاكرة التخزين المؤقت HTTP`_؛

    * `HTTP Cache في Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`مستندات Varnish`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Varnish HTTP Cache docs`: https://varnish-cache.org/docs/index.html
.. _`مواصفات ESI`: https://www.w3.org/TR/esi-lang
.. _`موارد مطوري ESI`: https://www.akamai.com/us/en/support/esi.jsp
.. _`نموذج التحقق من ذاكرة التخزين المؤقت HTTP`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP Cache في Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
