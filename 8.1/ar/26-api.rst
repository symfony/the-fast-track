عرض API مع منصة  (API Platform)
========================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

لقد انتهينا من تنفيذ موقع سجل الزوار. للسماح بمزيد من استخدام البيانات ، ماذا عن كشف API الآن؟ يمكن استخدام واجهة برمجة التطبيقات (API) بواسطة تطبيق جوال لعرض جميع المؤتمرات ، وتعليقاتهم ، وربما السماح للحاضرين بإرسال التعليقات.

في هذه الخطوة ، سنقوم بتنفيذ API للقراءة فقط.

تثبيت منصة API
-----------------------

من الممكن تعريض واجهة برمجة التطبيقات عن طريق كتابة بعض الأكواد ، ولكن إذا أردنا استخدام المعايير ، فمن الأفضل أن نستخدم حلاً يعتني بالفعل بالتحميل الثقيل. حل مثل API المنصة:

.. code-block:: terminal

    $ symfony composer req api

تعريض API للمؤتمرات
---------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

بعض السمات على فئة المؤتمر هي كل ما نحتاج إليه لتكوين API

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -2,29 +2,45 @@

     namespace App\Entity;

    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\ConferenceRepository;
     use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;
     use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;
    +use Symfony\Component\Serializer\Attribute\Groups;
     use Symfony\Component\String\Slugger\SluggerInterface;

     #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
     #[UniqueEntity('slug')]
    +#[ApiResource(
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'conference:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'conference:list'])
    +    ],
    +    order: ['year' => 'DESC', 'city' => 'ASC'],
    +    paginationEnabled: false,
    +)]
     class Conference
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $city = null;

         #[ORM\Column(length: 4)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $year = null;

         #[ORM\Column]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?bool $isInternational = null;

         /**
    @@ -34,6 +50,7 @@ class Conference
         private Collection $comments;

         #[ORM\Column(length: 255, unique: true)]
    +    #[Groups(['conference:list', 'conference:item'])]
         private ?string $slug = null;

         public function __construct()

تقوم السمة الرئيسية ``ApiResource`` بتكوين واجهة برمجة التطبيقات للمؤتمرات. يقيد العمليات المحتملة `` get``وتكوين أشياء مختلفة: مثل الحقول التي سيتم عرضها وكيفية ترتيب المؤتمرات.

بشكل افتراضي ، نقطة الدخول الرئيسية لواجهة برمجة التطبيقات هي ``/api`` بفضل التكوين من``config/routes/api_platform.yaml`` التي تمت إضافتها بواسطة وصفة الحزمة.

تسمح لك واجهة الويب بالتفاعل مع واجهة برمجة التطبيقات:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

استخدمها لاختبار الاحتمالات المختلفة:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

تخيل الوقت الذي يستغرقه تنفيذ كل هذا من الصفر!

تعريض API للتعليقات
---------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

افعل نفس الشيء للتعليقات:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -2,41 +2,63 @@

     namespace App\Entity;

    +use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
    +use ApiPlatform\Metadata\ApiFilter;
    +use ApiPlatform\Metadata\ApiResource;
    +use ApiPlatform\Metadata\Get;
    +use ApiPlatform\Metadata\GetCollection;
     use App\Repository\CommentRepository;
     use Doctrine\DBAL\Types\Types;
     use Doctrine\ORM\Mapping as ORM;
    +use Symfony\Component\Serializer\Attribute\Groups;
     use Symfony\Component\Validator\Constraints as Assert;

     #[ORM\Entity(repositoryClass: CommentRepository::class)]
     #[ORM\HasLifecycleCallbacks]
    +#[ApiResource(
    +    operations: [
    +        new Get(normalizationContext: ['groups' => 'comment:item']),
    +        new GetCollection(normalizationContext: ['groups' => 'comment:list'])
    +    ],
    +    order: ['createdAt' => 'DESC'],
    +    paginationEnabled: false,
    +)]
    +#[ApiFilter(SearchFilter::class, properties: ['conference' => 'exact'])]
     class Comment
     {
         #[ORM\Id]
         #[ORM\GeneratedValue]
         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?int $id = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $author = null;

         #[ORM\Column(type: Types::TEXT)]
         #[Assert\NotBlank]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $text = null;

         #[ORM\Column(length: 255)]
         #[Assert\NotBlank]
         #[Assert\Email]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $email = null;

         #[ORM\Column]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?\DateTimeImmutable $createdAt = null;

         #[ORM\ManyToOne(inversedBy: 'comments')]
         #[ORM\JoinColumn(nullable: false)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?Conference $conference = null;

         #[ORM\Column(length: 255, nullable: true)]
    +    #[Groups(['comment:list', 'comment:item'])]
         private ?string $photoFilename = null;

         #[ORM\Column(length: 255, options: ['default' => 'submitted'])]

يتم استخدام نفس النوع من السمات لتكوين الفصل.

تقييد التعليقات المكشوفة بواسطة API
---------------------------------------------------------------

بشكل افتراضي ، يكشف منصة API كل الإدخالات من قاعدة البيانات. ولكن بالنسبة للتعليقات ، يجب أن تكون التعليقات المنشورة فقط جزءًا من واجهة برمجة التطبيقات.

عندما تحتاج إلى تقييد العناصر التي يتم إرجاعها بواسطة API ، قم بإنشاء خدمة تنفذ ``QueryCollectionExtensionInterface`` للتحكم في استعلام Doctrine المستخدم في المجموعات و / أو ``QueryItemExtensionInterface`` للتحكم في العناصر:

.. code-block:: php
    :caption: src/Api/FilterPublishedCommentQueryExtension.php
    :emphasize-lines: 14-16,21-23

    namespace App\Api;

    use ApiPlatform\Doctrine\Orm\Extension\QueryCollectionExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Extension\QueryItemExtensionInterface;
    use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
    use ApiPlatform\Metadata\Operation;
    use App\Entity\Comment;
    use Doctrine\ORM\QueryBuilder;

    class FilterPublishedCommentQueryExtension implements QueryCollectionExtensionInterface, QueryItemExtensionInterface
    {
        public function applyToCollection(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }

        public function applyToItem(QueryBuilder $queryBuilder, QueryNameGeneratorInterface $queryNameGenerator, string $resourceClass, array $identifiers, Operation $operation = null, array $context = []): void
        {
            if (Comment::class === $resourceClass) {
                $queryBuilder->andWhere(sprintf("%s.state = 'published'", $queryBuilder->getRootAliases()[0]));
            }
        }
    }

تطبق فئة ملحق الاستعلام المنطق الخاص بها فقط لمورد ``Comment`` وتعديل أداة إنشاء استعلام العقيدة للنظر في التعليقات في حالة ``published`` فقط.

تكوين CORS
---------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

تبعًا للإعدادات الافتراضية ، فإن سياسة الأمان ذات الأصل الأصلي لعملاء HTTP الحديث تجعل استدعاء API من مجال آخر محظور. حزمة CORS ، المثبتة كجزء من ``composer req api``، يرسل رؤوس مشاركة المصادر المشتركة استنادًا إلى متغير البيئة `` CORS_ALLOW_ORIGIN``

تبعًا للإعدادات الافتراضية ، تسمح قيمته ، المعرّفة في `` .env`` ، بطلبات HTTP من ``localhost`` و``127.0.0.1`` على أي منفذ. عدّلها عندما يحتاج تطبيق مستضاف على نطاق آخر، مثل تطبيق جوال أو واجهة أمامية خارجية، إلى الاتصال بـ API.

.. sidebar:: الذهاب أبعد من ذلك

    * `SymfonyCasts API Platform tutorial`_؛

    * لتمكين دعم GraphQL ، قم بتشغيل ``composer require webonyx/graphql-php`` ، ثم استعرض للوصول إلى ``api/graphql/``.

.. _`SymfonyCasts API Platform tutorial`: https://symfonycasts.com/screencast/api-platform
