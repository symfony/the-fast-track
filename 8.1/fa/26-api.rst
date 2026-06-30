ارائه‌ی یک API با استفاده از API Platform
==============================================================

.. index::
    single: API
    single: HTTP API
    single: API Platform

ما پیاده‌سازی وب‌سایت Guestbook را تمام کرده‌ایم. برای اینکه اجازه دهیم از داده‌ها بیشتر استفاده شود، نظرتان در مورد ارائه‌ی یک API چیست؟ یک API می‌تواند توسط اپلیکیشن‌های موبایلی، برای نمایش تمام کنفرانس‌ها و کامنت‌هایشان مورد استفاده قرار بگیرد یا حتی به کاربران اجازه دهد تا کامنت ارسال کنند.

در این گام، می‌خواهیم یک API فقط‌خواندنی را پیاده‌سازی کنیم.

نصب API Platform
-------------------

ارائه‌ی یک API با نوشتن مقداری کد امکان‌پذیر است. اما اگر می‌خواهیم از استانداردها استفاده کنیم، بهتر است از راهکاری بهره بگیریم که بخش سخت کار را انجام دهد. راهکاری مثل API Platform:

.. code-block:: terminal

    $ symfony composer req api

ارائه‌ی یک API برای کنفرانس‌ها
-------------------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;Groups

تعدادی attribute بر روی کلاس Conference، تمام چیزی است که برای پیکربندی API احتیاج داریم:

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

attribute اصلی ``ApiResource``، API را برای کنفرانس‌ها پیکربندی می‌کند. این attribute عملیات‌های ممکن را به ``get`` محدود می‌کند و چیزهای مختلفی را پیکربندی می‌کند: همچون اینکه چه فیلدهایی نمایش داده شود و ترتیب کنفرانس‌ها به چه شکل باشد.

به صورت پیش‌فرض و به لطف پیکربندی موجود در ``config/routes/api_platform.yaml`` که توسط recipe‌ مربوط به بسته اضافه شده است، مدخل اصلی برای API همان ``/api`` است.

رابط وب به شما اجازه می‌ده تا با API فعل‌وانفعال داشته باشید:

.. figure:: screenshots/api.png
    :alt: /api
    :align: center
    :figclass: with-browser

از آن استفاده کنید تا امکانات مختلف را امتحان کنید:

.. figure:: screenshots/api-conferences.png
    :alt: /api
    :align: center
    :figclass: with-browser

تصور کنید که اگر تمام این‌ها را از ابتدا پیاده‌سازی می‌کردید چقدر طول می‌کشید!

ارائه‌ی API برای کامنت‌ها
----------------------------------------------

.. index::
    single: Attributes;ApiResource
    single: Attributes;ApiFilter
    single: Attributes;Groups

همین کار را برای کامنت‌ها بکنید:

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

از attributeهای مشابه‌ای برای پیکربندی کلاس استفاده شده است.

محدودسازی کامنت‌هایی که توسط API ارائه گردیده
----------------------------------------------------------------------------------

به صورت پیش‌فرض، API Platform تمام کامنت‌های درون پایگاه‌داده را ارائه می‌کند. اما برای کامنت‌ها، تنها باید آن‌هایی که منتشر‌شده هستند بخشی از API باشند.

زمانی که لازم دارید آیتم‌های بازگردانده‌شده توسط API را محدود کنید، سرویسی ایجاد کنید که یا رابط ``QueryCollectionExtensionInterface`` را که برای کنترل پرس‌وجو‌های Doctrine مربوط برای collectionها است، پیاده‌سازی کند یا اینکه رابط ``QueryItemExtensionInterface`` را پیاده‌سازی کند که برای کنترل آیتم‌ها مورد استفاده قرار می‌گیرد:

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

این کلاسِ بسط پرس‌وجو (query extension)، منطقش را تنها به منبع ``Comment`` اعمال می‌کند و سازنده‌ی پرس‌وجوی Doctrine را تغییر می‌دهد تا تنها کامنت‌هایی با وضعیت ``published`` را در نظر بگیرد.

پیکربندی CORS
---------------------

.. index::
    single: CORS
    single: Cross-Origin Resource Sharing

به صورت پیش‌فرض، در تمام کلاینت‌های مدرن HTTP، سیاست امنیتی same-origin، فراخوانی API از سایر دامنه‌ها را ممنوع می‌کند. باندل CORS، که به عنوان بخشی از ``composer req api`` نصب گردیده است، سربرگ Cross-Origin Resource Sharing را بر اساس متغیر محیط ``CORS_ALLOW_ORIGIN``، ارسال می‌کند.

به صورت پیش‌فرض، مقدار آن که در ``.env`` تعریف شده است، درخواست‌های HTTP از ``localhost`` و ``127.0.0.1`` را بر روی هر درگاهی  (port) اجازه می‌دهد. زمانی که یک اپلیکیشن میزبانی‌شده بر روی دامنه‌ای دیگر، مانند یک اپلیکیشن موبایل یا یک فرانت‌اند خارجی، نیاز به فراخوانی API داشته باشد، آن را تطبیق دهید.

.. sidebar:: بیشتر بدانید

    * `آموزش تصویری API Platform در SymfonyCasts`_؛

    * برای فعال‌سازی پشتیبانی از GraphQL، فرمان ``composer require webonyx/graphql-php`` را اجرا کنید و سپس آدرس ``/api/graphql`` را مرور کنید.

.. _`آموزش تصویری API Platform در SymfonyCasts`: https://symfonycasts.com/screencast/api-platform
