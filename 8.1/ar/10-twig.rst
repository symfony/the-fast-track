إنشاء واجهة المستخدم
======================================

.. index::
    single: Twig
    single: Templates

كل شيء الآن في مكانه لإنشاء الإصدار الأول من واجهة المستخدم للموقع. لن نجعلها جميلة. فقط شغالة الآن.

هل تتذكر ال escaping الذي قمنا به في وحدة التحكم من أجل بيضة عيد الفصح لتجنب المشاكل الأمنية؟ لن نستخدم PHP لقوالبنا لهذا السبب. بدلاً من ذلك ، سنستخدم Twig. بالإضافة إلى معالجة escaping المخرجات لنا ، فإن `Twig`_ يجلب الكثير من الميزات الرائعة التي سنستفيد منها ، مثل وراثة النماذج.

استعمال Twig للقوالب
----------------------------------

.. index::
    single: Twig;Layout
    single: Twig;block

جميع الصفحات في الموقع الإلكتروني ستشارك نفس *التصميم*. عند تثبيت Twig, المجلد ``templates/`` سيتم إنشاؤه تلقائيا و كذالك التصميم العينة في ``base.html.twig``.

.. code-block:: html+twig
    :caption: templates/base.html.twig
    :class: ignore

    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 128 128%22><text y=%221.2em%22 font-size=%2296%22>⚫️</text><text y=%221.3em%22 x=%220.2em%22 font-size=%2276%22 fill=%22%23fff%22>sf</text></svg>">
            {% block stylesheets %}
            {% endblock %}

            {% block javascripts %}
                {% block importmap %}{{ importmap('app') }}{% endblock %}
            {% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
        </body>
    </html>

يمكن أن يحدد التصميم عناصر  من نوع ``block``، وهي الأماكن التي تضيف فيها *قوالب فرعية*  التي *تمد* التصميم محتوياتها.

.. index::
    single: Twig;extends
    single: Twig;for

لنقم بإنشاء قالب لصفحة المشروع الرئيسية في ``templates/conference/index.html.twig``:

.. code-block:: html+twig
    :caption: templates/conference/index.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook{% endblock %}

    {% block body %}
        <h2>Give your feedback!</h2>

        {% for conference in conferences %}
            <h4>{{ conference }}</h4>
        {% endfor %}
    {% endblock %}

القالب *يمتد* ``base.html.twig`` ويعيد تعريف كتل ``title`` و ``body``.

.. index::
    single: Twig;Syntax

تشير العلامة ``{% %}`` في القالب إلى *التصرفات* و * البنية*.

يتم استخدام علامة ``{{}}`` *لعرض* شيء ما. يعرض ``{{Conference}}`` تمثيل المؤتمر (نتيجة استدعاء ``toString__`` على كائن ``Conference``).

إستخدام Twig في وحدة التحكم
----------------------------------------------

قم بتحديث وحدة التحكم لعرض قالب Twig:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,22 +2,19 @@

     namespace App\Controller;

    +use App\Repository\ConferenceRepository;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    +use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response(<<<EOF
    -            <html>
    -                <body>
    -                    <img src="/images/under-construction.gif" />
    -                </body>
    -            </html>
    -            EOF
    -        );
    +        return new Response($twig->render('conference/index.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]));
         }
     }

هناك الكثير مما يجري هنا.

لكي نتمكن من رسم قالب، نحتاج الي كائن بيئة الـ Twig (نقطة الدخول الرئيسية بـ Twig). لاحظ اننا نطلب نموذج Twig من خلال الاشارة له (type-hinting) في منهجية وحدة التحكم. سيمفوني ذكي بما فيه الكفاية ليعلم كيف يقوم بحقن الكائن الصحيح.

نحتاج أيضًا إلى مستودع بيانات المؤتمر للحصول على جميع المؤتمرات من قاعدة البيانات.

في وحدة التحكم ، تعرض طريقة ``()render`` القالب وتمرر مجموعة من المتغيرات إلى القالب. نقوم بتمرير قائمة كائنات ``Conference`` كمتغير ``conferences``.

وحدة التحكم هي فئة PHP قياسية. لا نحتاج حتى إلى تمديد فئة `` AbstractController `` إذا أردنا أن نكون صريحين بشأن تبعياتنا. يمكنك إزالته (ولكن لا تفعل ذلك ، حيث سنستخدم الاختصارات الرائعة التي يوفرها في الخطوات المستقبلية).

إنشاء الصفحة لمؤتمر(Conference)
------------------------------------------------

يجب أن يكون لكل مؤتمر(Conference) صفحة مخصصة لسرد تعليقاته. إن إضافة صفحة جديدة هي مسألة إضافة وحدة تحكم ، وتحديد مسار (route) لها ، وإنشاء القالب المرتبط بها.

أضف ``()show`` الى ``src/Controller/ConferenceController.php``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -2,6 +2,9 @@

     namespace App\Controller;

    +use App\Entity\Conference;
    +use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    @@ -17,4 +20,13 @@ final class ConferenceController extends AbstractController
                 'conferences' => $conferenceRepository->findAll(),
             ]));
         }
    +
    +    #[Route('/conference/{id}', name: 'conference')]
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    {
    +        return new Response($twig->render('conference/show.html.twig', [
    +            'conference' => $conference,
    +            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +        ]));
    +    }
     }

هذه الطريقة لها سلوك خاص لم نره بعد. نطلب أن يتم إدخال  نموذج `` Conference `` في الطريقة. ولكن قد يكون هناك العديد منها في قاعدة البيانات. تخبر السمة ``#[MapEntity]`` Symfony بجلب الصحيح بناءً على `` {id} `` الذي تم تمريره في مسار الطلب `` id `` هو المفتاح الأساسي لجدول `` conference  `` في قاعدة البيانات).

يمكن استرجاع التعليقات المتعلقة بالمؤتمر من خلال طريقة `` ()findBy  `` التي تأخذ المعايير كحجة أولى.

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;for
    single: Twig;if
    single: Twig;else
    single: Twig;asset
    single: Twig;format_datetime
    single: Twig;length

الخطوة الأخيرة هي إنشاء ملف ``templates/conference/show.html.twig``:

.. code-block:: html+twig
    :caption: templates/conference/show.html.twig

    {% extends 'base.html.twig' %}

    {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

    {% block body %}
        <h2>{{ conference }} Conference</h2>

        {% if comments|length > 0 %}
            {% for comment in comments %}
                {% if comment.photofilename %}
                    <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
                {% endif %}

                <h4>{{ comment.author }}</h4>
                <small>
                    {{ comment.createdAt|format_datetime('medium', 'short') }}
                </small>

                <p>{{ comment.text }}</p>
            {% endfor %}
        {% else %}
            <div>No comments have been posted yet for this conference.</div>
        {% endif %}
    {% endblock %}

في هذا النموذج ، نستخدم الترميز ``|`` لاستدعاء Twig *فلتر*. يقوم المرشح بتحويل القيمة. يُرجع ``comments | length`` عدد التعليقات و ``comment.createdAt|format_datetime('medium', 'short')`` لتنسيق التاريخ في تمثيل يمكن قراءته.

حاول الوصول إلى المؤتمر "الأول" عبر `` Conference/1/ `` ولاحظ الخطأ التالي:

.. figure:: screenshots/intl-twig-error.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

يأتي الخطأ من `` format_datetime `` فلتر لأنه ليس جزءًا من نواة Twig. تمنحك رسالة الخطأ تلميحًا حول ال package  الذي يجب تثبيته لإصلاح المشكلة:

.. code-block:: terminal

    $ symfony composer req "twig/intl-extra:^3"

الآن الصفحة تعمل بشكل جيد.

ربط الصفحات معا
----------------------------

.. index::
    single: Twig;Link
    single: Link

الخطوة الأخيرة لإنهاء نسختنا الأولى من واجهة المستخدم هي ربط صفحات المؤتمر من الصفحة الرئيسية:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -7,5 +7,8 @@

         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
    +        <p>
    +            <a href="/conference/{{ conference.id }}">View</a>
    +        </p>
         {% endfor %}
     {% endblock %}

لكن تحديد المسار الثابت فكرة سيئة لعدة أسباب. السبب الأكثر أهمية هو إذا قمت بتغيير المسار (من ``{conference/{id/`` إلى ``{conference/{id/`` على سبيل المثال) ، يجب تحديث جميع الروابط يدويًا.

.. index::
    single: Twig;path

بدلاً من ذلك ، استخدم  *وظيفة* ``path()``  ل Twig *واستخدم* اسم المسار*:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="/conference/{{ conference.id }}">View</a>
    +            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}

تنشئ وظيفة `` path() `` المسار إلى الصفحة باستخدام اسم المسار الخاص بها. يتم تمرير قيم معلمات المسار (route parameters) كخريطة Twig.

ترقيم صفحات التعليقات
----------------------------------------

.. index::
    single: Doctrine;Paginator
    single: Paginator

بوجود الآلاف من الحاضرين ، يمكننا أن نتوقع بعض التعليقات. إذا عرضناها جميعًا على صفحة واحدة ، فسوف تنمو بسرعة كبيرة.

انشئ دالة ``getCommentPaginator()`` في مستودع التعليقات (Comment Repository) الذي يُرجع تعليق منسق صفحات استناداً الي مؤتمر ونقطة بداية (offset) (أين تبدأ):

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -3,19 +3,37 @@
     namespace App\Repository;

     use App\Entity\Comment;
    +use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
      * @extends ServiceEntityRepository<Comment>
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    public const COMMENTS_PER_PAGE = 2;
    +
         public function __construct(ManagerRegistry $registry)
         {
             parent::__construct($registry, Comment::class);
         }

    +    public function getCommentPaginator(Conference $conference, int $offset): Paginator
    +    {
    +        $query = $this->createQueryBuilder('c')
    +            ->andWhere('c.conference = :conference')
    +            ->setParameter('conference', $conference)
    +            ->orderBy('c.createdAt', 'DESC')
    +            ->setMaxResults(self::COMMENTS_PER_PAGE)
    +            ->setFirstResult($offset)
    +            ->getQuery()
    +        ;
    +
    +        return new Paginator($query);
    +    }
    +
         //    /**
         //     * @return Comment[] Returns an array of Comment objects
         //     */

لقد قمنا بتحديد الحد الأقصى لعدد التعليقات في الصفحة ل  2 لتسهيل الاختبار.

لإدارة ترقيم الصفحات في النموذج ، قم بتمرير Doctrine Paginator بدلاً من Doctrine Collection إلى Twig:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,6 +8,7 @@ use App\Repository\ConferenceRepository;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
     use Twig\Environment;

    @@ -22,11 +23,15 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository): Response
    +    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
    +        $paginator = $commentRepository->getCommentPaginator($conference, $offset);
    +
             return new Response($twig->render('conference/show.html.twig', [
                 'conference' => $conference,
    -            'comments' => $commentRepository->findBy(['conference' => $conference], ['createdAt' => 'DESC']),
    +            'comments' => $paginator,
    +            'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
    +            'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
             ]));
         }
     }

تربط السمة ``#[MapQueryParameter]`` معامل سلسلة الاستعلام ``offset`` بوسيط وحدة التحكم ``$offset``، مع جعل القيمة الافتراضية ``0`` عندما لا يكون مُعيّنًا. وبما أن الإزاحة تأتي من العميل، فإن الخيار ``min_range`` يتحقق من أنها ليست سالبة؛ ويُرجع Symfony استجابة 404 عندما تكون القيمة غير صالحة.

يتم حساب الإزاحات `` previous  `` و `` next `` بناءً على جميع المعلومات التي لدينا من paginator.

.. index::
    single: Twig;if

أخيرًا ، قم بتحديث القالب لإضافة روابط إلى الصفحات التالية والسابقة:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -6,6 +6,8 @@
         <h2>{{ conference }} Conference</h2>

         {% if comments|length > 0 %}
    +        <div>There are {{ comments|length }} comments.</div>
    +
             {% for comment in comments %}
                 {% if comment.photofilename %}
                     <img src="{{ asset('uploads/photos/' ~ comment.photofilename) }}" style="max-width: 200px" />
    @@ -18,6 +20,13 @@

                 <p>{{ comment.text }}</p>
             {% endfor %}
    +
    +        {% if previous >= 0 %}
    +            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +        {% endif %}
    +        {% if next < comments|length %}
    +            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +        {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>
         {% endif %}

يجب أن تكون قادرًا الآن على التنقل بين التعليقات عبر رابطي "Previous" و "Next":

.. figure:: screenshots/pagination-next.png
    :alt: /conference/1
    :align: center
    :figclass: with-browser

.. figure:: screenshots/pagination-previous.png
    :alt: /conference/1?offset=2
    :align: center
    :figclass: with-browser

إعادة هيكلة وحدة التحكم
-------------------------------------------

ربما لاحظت أن كلا من الطريقتين في `` ConferenceController `` تأخذان بيئة Twig كخاصية. بدلاً من حقنها في كل طريقة ، دعنا نستفيد من الطريقة المساعدة ``render()`` التي توفرها الفئة الأم:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,28 +9,27 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\Routing\Attribute\Route;
    -use Twig\Environment;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(Environment $twig, ConferenceRepository $conferenceRepository): Response
    +    public function index(ConferenceRepository $conferenceRepository): Response
         {
    -        return new Response($twig->render('conference/index.html.twig', [
    +        return $this->render('conference/index.html.twig', [
                 'conferences' => $conferenceRepository->findAll(),
    -        ]));
    +        ]);
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(Environment $twig, #[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

    -        return new Response($twig->render('conference/show.html.twig', [
    +        return $this->render('conference/show.html.twig', [
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,
                 'next' => min(count($paginator), $offset + CommentRepository::COMMENTS_PER_PAGE),
    -        ]));
    +        ]);
         }
     }

.. sidebar:: الذهاب أبعد من ذلك

    * `مستندات Twig`_؛

    * `إنشاء واستخدام قوالب`_ في تطبيقات Symfony؛

    * `البرنامج التعليمي SymfonyCasts Twig`_؛

    * `وظائف ومرشحات Twig المتوفرة فقط في Symfony`_؛

    * وحدة التحكم الأساسية `AbstractController`_.

.. _`Twig`: https://twig.symfony.com/
.. _`مستندات Twig`: https://twig.symfony.com/doc/3.x/
.. _`إنشاء واستخدام قوالب`: https://symfony.com/doc/current/templates.html
.. _`البرنامج التعليمي SymfonyCasts Twig`: https://symfonycasts.com/screencast/symfony/twig-recipe
.. _`وظائف ومرشحات Twig المتوفرة فقط في Symfony`: https://symfony.com/doc/current/reference/twig_reference.html
.. _`AbstractController`: https://symfony.com/doc/current/controller.html#the-base-controller-classes-services
