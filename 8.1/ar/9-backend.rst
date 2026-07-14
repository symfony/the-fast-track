إعداد  النظام الخلفي
=====================================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

تعتبر إضافة المؤتمرات القادمة إلى قاعدة البيانات مهمة خاصة بالمشرفين. *قاعدة الإشراف الخلفية* تعتبر جناح محمي فالموقع, حيث يمكن *للمشرفين* إدارة بيانات موقع الويب وتنسيق عمليات إرسال التعليقات وغير ذلك.

كيف يمكننا بداية العمل بسرعة ؟ باستعمال أدوات لإنشاء قاعدة الإشراف الخلفية، EasyAdmin يمكننا من القيام بهذه العملية بطريقة جيدة جدا.

تثبيت المزيد من التبعيات
-------------------------------------------------

رغم أن حزمة ``webapp`` أضافت تلقائيًا العديد من الحزم اللطيفة، إلا أننا نحتاج لبعض الميزات الأكثر تحديدًا إلى إضافة المزيد من التبعيات. كيف نضيف المزيد من التبعيات؟ عن طريق مؤلف Composer. بالاضافة الي حزمة كومبوزر Composer "الاساسية" ("regular")، سوف نقوم بالعمل مع نوعين "خاصين" من الحزم:

* *مكونات سيمفوني*: الحزم التي تنفذ الصفات الاساسية وعمليات منخفضة المستوي الي تحتاجها معظم البرامج (routing, console, HTTP client, mailer, cache, ...)؛

* *رزم سيمفوني*: حزم تقوم بإضافة مميزات عالية المستوي أو توفير التكامل مع المكتبات الخارجية (رزم يتم مشاركتها في الغالب بواسطة المجتمع).

دعنا نضيف EasyAdmin كتبعية للمشروع:

.. code-block:: terminal

    $ symfony composer req "easycorp/easyadmin-bundle:^5"

*الاسماء المستعارة (Aliases)* لا تعتبر خاصية كمبوزر، ولكن الفكرة مقدمة بواسطة سيمفوني لجعل حياتك أسهل. الاسماء المستعارة هي اختصارات للحزم الشائعة في كموزر. تريد ORM للتطبيق الخاص بك؟ إستدعي ``orm``. تريد ان تقوم ببناء API؟ أستدعي ``api``. يتم حل هذه الاسماء المستعارة بشكل تلقائي لواحدة او اكثر من حزم كمبوزر. إنها اختيارات مُقررة بواسطة فريق سيمفوني الاساسي.

ومن المميزات الانيقة الاخري انه يمكنك حزف مورد ``سيمفوني`` (``symfony`` vendor). إستدعي ``cache`` بدلاً من ``symfony/cache``.

.. tip::

    هل تتذكر بأننا قمنا بذكر مكون إضافي في composer يسمي ``symfony/flex``؟ الاسماء المستعارة واحدة من مميزاتها.

تهيئة EasyAdmin
--------------------

يقوم EasyAdmin تلقائيًا بإنشاء منطقة إدارة لتطبيقك بناءً على وحدات تحكم محددة.

للبدء مع EasyAdmin، دعنا نولّد "لوحة تحكم إدارة الويب" التي ستكون نقطة الدخول الرئيسية لإدارة بيانات الموقع:

.. code-block:: terminal
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

قبول الإجابات الافتراضية يُنشئ وحدة التحكم التالية:

.. code-block:: php
    :caption: src/Controller/Admin/DashboardController.php
    :class: ignore

    namespace App\Controller\Admin;

    use EasyCorp\Bundle\EasyAdminBundle\Attribute\AdminDashboard;
    use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
    use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
    use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    use Symfony\Component\HttpFoundation\Response;

    #[AdminDashboard(routePath: '/admin', routeName: 'admin')]
    class DashboardController extends AbstractDashboardController
    {
        public function index(): Response
        {
            return parent::index();
        }

        public function configureDashboard(): Dashboard
        {
            return Dashboard::new()
                ->setTitle('Guestbook');
        }

        public function configureMenuItems(): iterable
        {
            yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
            // yield MenuItem::linkTo(SomeCrudController::class, 'The Label', 'fas fa-list');
        }
    }

حسب الاصطلاح ، يتم تخزين جميع وحدات التحكم الإدارية ضمن مساحة الاسم الخاصة بها  ``App\Controller\Admin``

قم بالوصول الي خلفية المسؤول التي تم إنشائها في `` /admin`` كما هو مُهيأ بواسطة السمة ``#[AdminDashboard]``؛ يمكنك تغيير الرابط إلى أي شيء تريده:

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

فقاعة! لدينا واجهة مشرف لطيفة المظهر ، جاهزة للتخصيص حسب احتياجاتنا.

.. index::
    single: CRUD

الخطوة التالية هي إنشاء وحدات تحكم لإدارة المؤتمرات والتعليقات.

في وحدة التحكم في لوحة القيادة ، ربما تكون قد لاحظت طريقة ``configureMenuItems()`` التي تحتوي على تعليق حول إضافة روابط إلى "CRUDs". **CRUD** هي اختصار لعبارة "إنشاء وقراءة وتحديث وحذف" ، العمليات الأساسية الأربع التي تريد القيام بها على أي كيان. هذا هو بالضبط ما نريد أن يقوم به المسؤول لنا ؛ حتى أن EasyAdmin ينتقل إلى المستوى التالي من خلال الاهتمام أيضًا بالبحث والتصفية.

دعونا ننشئ CRUD للمؤتمرات:

.. code-block:: terminal
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

حدد ``1`` لإنشاء واجهة إدارة للمؤتمرات واستخدام الإعدادات الافتراضية للأسئلة الأخرى. يجب إنشاء الملف التالي:

.. code-block:: php
    :caption: src/Controller/Admin/ConferenceCrudController.php
    :class: ignore

    namespace App\Controller\Admin;

    use App\Entity\Conference;
    use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;

    class ConferenceCrudController extends AbstractCrudController
    {
        public static function getEntityFqcn(): string
        {
            return Conference::class;
        }

        /*
        public function configureFields(string $pageName): iterable
        {
            return [
                IdField::new('id'),
                TextField::new('title'),
                TextEditorField::new('description'),
            ];
        }
        */
    }

افعل الشيء نفسه للتعليقات:

.. code-block:: terminal
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

الخطوة الأخيرة هي ربط المؤتمر وتعليقات مشرف CRUDs بلوحة القيادة:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/DashboardController.php
    +++ w/src/Controller/Admin/DashboardController.php
    @@ -44,7 +44,8 @@ class DashboardController extends AbstractDashboardController

         public function configureMenuItems(): iterable
         {
    -        yield MenuItem::linkToDashboard('Dashboard', 'fa fa-home');
    -        // yield MenuItem::linkTo(SomeCrudController::class, 'The Label', 'fas fa-list');
    +        yield MenuItem::linkToRoute('Back to the website', 'fas fa-home', 'homepage');
    +        yield MenuItem::linkTo(ConferenceCrudController::class, 'Conferences', 'fas fa-map-marker-alt');
    +        yield MenuItem::linkTo(CommentCrudController::class, 'Comments', 'fas fa-comments');
         }
     }

لقد تجاوزنا طريقة ``configureMenuItems()`` لإضافة عناصر القائمة مع الرموز ذات الصلة بالمؤتمرات وأيقونات التعليقات ولإضافة ارتباط إلى الصفحة الرئيسية للموقع. تعيش فئتا ``ConferenceCrudController`` و ``CommentCrudController`` في نفس مساحة الاسم ``App\Controller\Admin`` مثل لوحة القيادة، لذا لا تحتاجان إلى عبارات ``use`` إضافية.

يعرض EasyAdmin واجهة برمجة تطبيقات لتسهيل الارتباط بالكيان CRUDS عبر طريقة ``MenuItem::linkTo()`` التي تأخذ فئة وحدة تحكم CRUD.

الصفحة الرئيسية للوحة القيادة فارغة في الوقت الحالي. هذا هو المكان الذي يمكنك فيه عرض بعض الإحصائيات أو أي معلومات ذات صلة. نظرًا لأنه ليس لدينا أي شيء مهم لعرضه ، فلنقم بإعادة التوجيه إلى قائمة المؤتمرات:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/DashboardController.php
    +++ w/src/Controller/Admin/DashboardController.php
    @@ -8,6 +8,7 @@ use EasyCorp\Bundle\EasyAdminBundle\Attribute\AdminDashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\Dashboard;
     use EasyCorp\Bundle\EasyAdminBundle\Config\MenuItem;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractDashboardController;
    +use EasyCorp\Bundle\EasyAdminBundle\Router\AdminUrlGenerator;
     use Symfony\Component\HttpFoundation\Response;

     #[AdminDashboard(routePath: '/admin', routeName: 'admin')]
    @@ -15,7 +16,10 @@ class DashboardController extends AbstractDashboardController
     {
         public function index(): Response
         {
    -        return parent::index();
    +        $routeBuilder = $this->container->get(AdminUrlGenerator::class);
    +        $url = $routeBuilder->setController(ConferenceCrudController::class)->generateUrl();
    +
    +        return $this->redirect($url);

             // Option 1. You can make your dashboard redirect to some common page of your backend
             //

عند عرض علاقات الكيانات (المؤتمر المرتبط بتعليق) ، يحاول EasyAdmin استخدام تمثيل سلسلة للمؤتمر. بشكل افتراضي ، يستخدم اصطلاحًا يستخدم اسم الكيان والمفتاح الأساسي ( مثل ``Conference #1``) إذا لم يحدد الكيان "magic" طريقة ``__toString()`` . لجعل العرض أكثر وضوحًا ، أضف هذه الطريقة إلى فئة ``Conference``:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -35,6 +35,11 @@ class Conference
             $this->comments = new ArrayCollection();
         }

    +    public function __toString(): string
    +    {
    +        return $this->city.' '.$this->year;
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;

يمكنك الآن إضافة / تعديل / حذف المؤتمرات مباشرة من الواجهة الخلفية للمشرف. العب بها وأضف مؤتمرًا واحدًا على الأقل.

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

تخصيص EasyAdmin
--------------------

تعمل الواجهة الخلفية الافتراضية للمشرف بشكل جيد ، ولكن يمكن تخصيصها بعدة طرق لتحسين التجربة. دعونا نقوم ببعض التغييرات البسيطة على كيان Comment لتوضيح بعض الاحتمالات:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -3,10 +3,17 @@
     namespace App\Controller\Admin;

     use App\Entity\Comment;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Crud;
    +use EasyCorp\Bundle\EasyAdminBundle\Config\Filters;
     use EasyCorp\Bundle\EasyAdminBundle\Controller\AbstractCrudController;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\AssociationField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\DateTimeField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\EmailField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\IdField;
    +use EasyCorp\Bundle\EasyAdminBundle\Field\TextareaField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextEditorField;
     use EasyCorp\Bundle\EasyAdminBundle\Field\TextField;
    +use EasyCorp\Bundle\EasyAdminBundle\Filter\EntityFilter;

     class CommentCrudController extends AbstractCrudController
     {
    @@ -15,14 +22,43 @@ class CommentCrudController extends AbstractCrudController
             return Comment::class;
         }

    -    /*
    +    public function configureCrud(Crud $crud): Crud
    +    {
    +        return $crud
    +            ->setEntityLabelInSingular('Conference Comment')
    +            ->setEntityLabelInPlural('Conference Comments')
    +            ->setSearchFields(['author', 'text', 'email'])
    +            ->setDefaultSort(['createdAt' => 'DESC'])
    +        ;
    +    }
    +
    +    public function configureFilters(Filters $filters): Filters
    +    {
    +        return $filters
    +            ->add(EntityFilter::new('conference'))
    +        ;
    +    }
    +
         public function configureFields(string $pageName): iterable
         {
    -        return [
    -            IdField::new('id'),
    -            TextField::new('title'),
    -            TextEditorField::new('description'),
    -        ];
    +        yield AssociationField::new('conference');
    +        yield TextField::new('author');
    +        yield EmailField::new('email');
    +        yield TextareaField::new('text')
    +            ->hideOnIndex()
    +        ;
    +        yield TextField::new('photoFilename')
    +            ->onlyOnIndex()
    +        ;
    +
    +        $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
    +            'years' => range(date('Y'), date('Y') + 5),
    +            'widget' => 'single_text',
    +        ]);
    +        if (Crud::PAGE_EDIT === $pageName) {
    +            yield $createdAt->setFormTypeOption('disabled', true);
    +        } else {
    +            yield $createdAt;
    +        }
         }
    -    */
     }

لتخصيص قسم ``Comment`` ، يتيح لنا إدراج الحقول بشكل صريح في طريقة ``configureFields()`` ترتيبها بالطريقة التي نريدها. يتم تكوين بعض الحقول بشكل أكبر ، مثل إخفاء حقل النص في صفحة الفهرس.

أضف بعض التعليقات بدون صور. قم بوضع التاريخ يدوياً في الوقت الحالي؛ سوف نقوم بملئ خلية الـ ``createdAt`` تلقائياً في خطوة مُتقدمة.

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

تحدد طريقة ``configureFilters()`` المرشحات التي يجب عرضها أعلى حقل البحث.

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

هذه التخصيصات هي مجرد مقدمة صغيرة للإمكانيات التي يوفرها EasyAdmin.

العب مع المسؤول ، وقم بتصفية التعليقات حسب المؤتمر ، أو ابحث عن التعليقات عبر البريد الإلكتروني على سبيل المثال. المشكلة الوحيدة هي أنه يمكن لأي شخص الوصول إلى الواجهة الخلفية. لا تقلق ، سنقوم بتأمينه في خطوة مستقبلية.

.. code-block:: terminal
    :class: hide

    $ symfony console dbal:run-sql "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: الذهاب أبعد من ذلك

    * `EasyAdmin توثيق`_؛

    * `Symfony framework مرجع الإعدادات`_؛

    * `طرق PHP السحرية`_.

.. _`EasyAdmin توثيق`: https://symfony.com/bundles/EasyAdminBundle/4.x/index.html
.. _`Symfony framework مرجع الإعدادات`: https://symfony.com/doc/current/reference/configuration/framework.html
.. _`طرق PHP السحرية`: https://www.php.net/manual/en/language.oop5.magic.php
