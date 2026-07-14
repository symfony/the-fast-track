إدارة دورة حياة  Doctrine Objects
==============================================

عند إنشاء تعليق جديد ، سيكون أمر رائع أن يتم تعيين تاريخ `` createAt `` تلقائيًا بقيمة التاريخ والوقت الحاليين.

لدى Doctrine طرق مختلفة للتعامل مع الكائنات وخصائصها أثناء دورة حياتها (قبل إنشاء الصف في قاعدة البيانات ، بعد تحديث الصف ، ...).

تحديد الاسترجاعات لدورة الحياة
---------------------------------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

عندما لا يحتاج السلوك إلى أي خدمة ويجب تطبيقه على نوع واحد فقط من الكيان ، حدد رد الاتصال (callback) في فئة الكيان:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -57,8 +57,6 @@ class CommentCrudController extends AbstractCrudController
             ]);
             if (Crud::PAGE_EDIT === $pageName) {
                 yield $createdAt->setFormTypeOption('disabled', true);
    -        } else {
    -            yield $createdAt;
             }
         }
     }
    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -7,6 +7,7 @@ use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
    +#[ORM\HasLifecycleCallbacks]
     class Comment
     {
         #[ORM\Id]
    @@ -86,6 +87,12 @@ class Comment
             return $this;
         }

    +    #[ORM\PrePersist]
    +    public function setCreatedAtValue(): void
    +    {
    +        $this->createdAt = new \DateTimeImmutable();
    +    }
    +
         public function getConference(): ?Conference
         {
             return $this->conference;

يتم تشغيل *حدث* ``ORM \ PrePersist``  عندما يتم تخزين الكائن في قاعدة البيانات لأول مرة. عند حدوث ذلك ، يتم استدعاء طريقة ``()setCreatedAtValue`` ويتم استخدام التاريخ والوقت الحاليين كقيمة لخاصية ``createdAt``.

إضافة البزاقات إلى المؤتمرات
-----------------------------------------------------

عناوين URL للمؤتمرات ليست ذات معنى: ``Conference/1/``. والأهم أنها تعتمد على تفاصيل التنفيذ (المفتاح الأساسي في قاعدة البيانات مسرب).

ماذا عن استخدام عناوين URL مثل ``/conference/paris-2020`` بدلاً من ذلك؟ هذا سيبدو أفضل بكثير. ``paris-2020`` هو ما نسميه المؤتمر *slug*.

.. index::
    single: Command;make:entity

قم بإضافة خاصية `` slug `` جديدة للمؤتمرات (سلسلة غير قابلة للإلغاء من 255 حرفًا):

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

قم بإنشاء ملف ترحيل لإضافة العمود الجديد:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

وتنفيذ هذا الترحيل الجديد:

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

هل حصلت على خطأ؟ هذا أمر متوقع. لماذا ا؟ لأننا طلبنا ألا يكون slug `` null `` ولكن الإدخالات الموجودة في قاعدة بيانات المؤتمر ستحصل على قيمة `` null `` عند تشغيل الترحيل. لنصلح ذلك من خلال تعديل الترحيل:

.. code-block:: diff
    :caption: patch_file

    --- i/migrations/Version00000000000000.php
    +++ w/migrations/Version00000000000000.php
    @@ -20,7 +20,9 @@ final class Version00000000000000 extends AbstractMigration
         public function up(Schema $schema): void
         {
             // this up() migration is auto-generated, please modify it to your needs
    -        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255) NOT NULL');
    +        $this->addSql('ALTER TABLE conference ADD slug VARCHAR(255)');
    +        $this->addSql("UPDATE conference SET slug=CONCAT(LOWER(city), '-', year)");
    +        $this->addSql('ALTER TABLE conference ALTER COLUMN slug SET NOT NULL');
         }

         public function down(Schema $schema): void

الحيلة هنا هي إضافة العمود والسماح له بأن يكون ``null`` ، ثم قم بتعيين slug إلى قيمة ليست ``null`` ، وأخيرًا ، قم بتغيير عمود slug لعدم السماح بـ ``null``.

.. note::

    بالنسبة لمشروع حقيقي ، قد يكون استخدام ``CONCAT(LOWER(city), '-', year)`` غير كافٍ. في هذه الحالة ، نحتاج إلى استخدام الSlugger "الحقيقي".

.. index::
    single: Command;doctrine:migrations:migrate

يجب أن يتم الترحيل بشكل جيد الآن:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

نظرًا لأن التطبيق سيستخدم slugs قريبًا للعثور على كل مؤتمر ، فلنقم بتعديل كيان المؤتمر للتأكد من أن قيم slug فريدة في قاعدة البيانات:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -6,8 +6,10 @@ use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    +#[UniqueEntity('slug')]
     class Conference
     {
         #[ORM\Id]
    @@ -30,7 +32,7 @@ class Conference
         #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
         private Collection $comments;

    -    #[ORM\Column(length: 255)]
    +    #[ORM\Column(length: 255, unique: true)]
         private ?string $slug = null;

         public function __construct()

.. index::
    single: Command;make:migration

كما كنت قد خمنت ، نحتاج إلى أداء رقصة الترحيل:

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

توليد البزاقات
---------------------------

.. index::
    single: Components;String
    single: Slug

يعد إنشاء سبيكة تقرأ جيدًا في عنوان URL (حيث يجب ترميز أي شيء بخلاف أحرف ASCII) مهمة صعبة ، خاصة للغات الأخرى غير الإنجليزية. كيف يمكنك تحويل "é" إلى "e" على سبيل المثال؟

بدلاً من إعادة اختراع العجلة ، دعنا نستخدم مكون  ``String`` ل Symfony، الذي يخفف من معالجة السلاسل strings  ويوفر *slugger*.

أضف طريقة `` ()computeSlug `` إلى فئة `` المؤتمر `` التي تحسب البزاق بناءً على بيانات المؤتمر:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -7,6 +7,7 @@ use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    @@ -50,6 +51,13 @@ class Conference
             return $this->id;
         }

    +    public function computeSlug(SluggerInterface $slugger): void
    +    {
    +        if (!$this->slug || '-' === $this->slug) {
    +            $this->slug = (string) $slugger->slug((string) $this)->lower();
    +        }
    +    }
    +
         public function getCity(): ?string
         {
             return $this->city;

لا تحسب طريقة `` ()computeSlug `` بزاق إلا إذا كان السلق الحالي فارغًا أو تم تعيينه على القيمة "-" الخاصة. لماذا نحتاج إلى القيمة الخاصة "-"؟ لأنه عند إضافة مؤتمر في الخلفية ، يكون البزاق مطلوبًا. لذا ، نحتاج إلى قيمة غير فارغة تخبر التطبيق أننا نريد إنشاء بزاق بشكل تلقائي.

تحديد استدعاء دورة حياة معقدة
------------------------------------------------------

.. index::
    single: Doctrine;Entity Listener

بالنسبة لخاصية ``createdAt``، يجب ضبط ``slug`` تلقائيًا عندما يتم تحديث المؤتمر عن طريق استدعاء طريقة ``()computeSlug``.

ولكن نظرًا لأن هذه الطريقة تعتمد على تطبيق ``SluggerInterface``، لا يمكننا إضافة حدث ``prePersist`` كما فعلنا (ليس لدينا طريقة لإدخال slugger).

بدلاً من ذلك ، قم بإنشاء مستمع كيان ل Doctrine:

.. code-block:: php
    :caption: src/EntityListener/ConferenceEntityListener.php

    namespace App\EntityListener;

    use App\Entity\Conference;
    use Doctrine\ORM\Event\PrePersistEventArgs;
    use Doctrine\ORM\Event\PreUpdateEventArgs;
    use Symfony\Component\String\Slugger\SluggerInterface;

    class ConferenceEntityListener
    {
        public function __construct(
            private SluggerInterface $slugger,
        ) {
        }

        public function prePersist(Conference $conference, PrePersistEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }

        public function preUpdate(Conference $conference, PreUpdateEventArgs $event): void
        {
            $conference->computeSlug($this->slugger);
        }
    }

لاحظ أن الslug يتم تحديثه عند إنشاء مؤتمر جديد (``()prePersist``) وكلما تم تحديثه (``()preUpdate``).

تكوين الخدمة في الحاوية
-------------------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

حتى الآن ، لم نتحدث عن أحد المكونات الرئيسية في Symfony ، *حاوية حقن التبعية (dependency injection container)*. الحاوية مسؤولة عن إدارة *الخدمات*: إنشائها وحقنها عند الحاجة.

*الخدمة* عبارة عن كائن "عام (global)" يوفر ميزات (على سبيل المثال ، مرسل بريد أو مسجّل أو سلوجر وما إلى ذلك) على عكس *كائنات البيانات (data objects)* (مثل مثيلات الكيان ل Doctrine).

نادرًا ما تتفاعل مع الحاوية مباشرة حيث تقوم تلقائيًا بإدخال كائنات الخدمة كلما احتجت إليها: تقوم الحاوية بحقن كائنات وسيطة وحدة التحكم عندما تكتب تلميحًا لها على سبيل المثال.

إذا تساءلت عن كيفية تسجيل مستمع الأحداث في الخطوة السابقة ، فلديك الجواب الآن: الحاوية. عندما يقوم الفصل بتنفيذ بعض الواجهات المحددة ، تعرف الحاوية أنه يجب تسجيل الفصل بطريقة معينة.

هنا ، نظرًا لأن فئتنا لا تطبق أي واجهة ولا توسع أي فئة أساسية ، لا تعرف Symfony كيفية إعدادها تلقائيًا. بدلاً من ذلك ، يمكننا استخدام سمة لإخبار حاوية Symfony بكيفية ربطها:

.. code-block:: diff
    :caption: patch_file

    --- i/src/EntityListener/ConferenceEntityListener.php
    +++ w/src/EntityListener/ConferenceEntityListener.php
    @@ -3,10 +3,14 @@
     namespace App\EntityListener;

     use App\Entity\Conference;
    +use Doctrine\Bundle\DoctrineBundle\Attribute\AsEntityListener;
     use Doctrine\ORM\Event\PrePersistEventArgs;
     use Doctrine\ORM\Event\PreUpdateEventArgs;
    +use Doctrine\ORM\Events;
     use Symfony\Component\String\Slugger\SluggerInterface;

    +#[AsEntityListener(event: Events::prePersist, entity: Conference::class)]
    +#[AsEntityListener(event: Events::preUpdate, entity: Conference::class)]
     class ConferenceEntityListener
     {
         public function __construct(

.. note::

    لا تخلط بين مستمعي حدث Doctrine  و Symfony. حتى لو بدوا متشابهين جدًا ، فإنهم لا يستخدمون نفس البنية التحتية تحت الغطاء.

استخدام البزاقات في التطبيق
---------------------------------------------------

حاول إضافة المزيد من المؤتمرات في الخلفية وتغيير المدينة أو سنة مؤتمر معروف ؛ لن يتم تحديث slug إلا إذا كنت تستخدم القيمة "-" الخاصة.

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

التغيير الأخير هو تحديث وحدات التحكم والقوالب لاستخدام المؤتمر `` slug `` بدلاً من `` id `` الخاص بالمؤتمر للمسارات. بما أن معامل المسار لم يعد المفتاح الأساسي للكيان ، استخدم الصيغة ``{slug:conference}`` لإخبار Symfony بجلب ``$conference`` عن طريق مطابقة خاصية ``slug`` الخاصة به؛ لم تعد السمة ``#[MapEntity]`` ضرورية:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -5,7 +5,6 @@
     use App\Entity\Conference;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    @@ -20,6 +20,6 @@ final class ConferenceController extends AbstractController
             ]);
         }

    -    #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
    +    #[Route('/conference/{slug:conference}', name: 'conference')]
    +    public function show(Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -16,7 +16,7 @@
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
                 <ul>
                 {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
                 {% endfor %}
                 </ul>
                 <hr />
    --- i/templates/conference/index.html.twig
    +++ w/templates/conference/index.html.twig
    @@ -8,7 +8,7 @@
         {% for conference in conferences %}
             <h4>{{ conference }}</h4>
             <p>
    -            <a href="{{ path('conference', { id: conference.id }) }}">View</a>
    +            <a href="{{ path('conference', { slug: conference.slug }) }}">View</a>
             </p>
         {% endfor %}
     {% endblock %}
    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -22,10 +22,10 @@
             {% endfor %}

             {% if previous >= 0 %}
    -            <a href="{{ path('conference', { id: conference.id, offset: previous }) }}">Previous</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: previous }) }}">Previous</a>
             {% endif %}
             {% if next < comments|length %}
    -            <a href="{{ path('conference', { id: conference.id, offset: next }) }}">Next</a>
    +            <a href="{{ path('conference', { slug: conference.slug, offset: next }) }}">Next</a>
             {% endif %}
         {% else %}
             <div>No comments have been posted yet for this conference.</div>

يجب أن يتم الوصول إلى صفحات المؤتمر الآن من خلال البزاق الخاص به:

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: الذهاب أبعد من ذلك

    * `نظام أحداث Doctrine`_ (استدعاءات دورة الحياة والمستمعين ، ومستمعي الكيانات ومشتركي دورة الحياة)؛

    * `مستندات مكون String`_؛

    * `حاوية الخدمة Service container`_؛

    * ال `Symfony Services Cheat Sheet`_.

.. _`نظام أحداث Doctrine`: https://symfony.com/doc/current/doctrine/events.html
.. _`مستندات مكون String`: https://symfony.com/doc/current/components/string.html
.. _`حاوية الخدمة Service container`: https://symfony.com/doc/current/service_container.html
.. _`Symfony Services Cheat Sheet`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
