توصیف ساختار داده
================================

.. index::
    single: Doctrine
    single: Database

برای کار کردن با پایگاه‌داده از طریق PHP، ما تصمیم داریم بر `Doctrine`_ تکیه کنیم. Doctrine مجموعه‌ای از کتابخانه‌ها است که برای مدیریت پایگاه‌داده‌ها به توسعه‌دهنده‌گان کمک می‌کند: Doctrine DBAL (یک لایه‌ی انتزاعی از پایگاه‌داده)، Doctrine ORM (یک کتابخانه برای دستکاری محتوای پایگاه‌داده از طریق اشیاء PHP) و Doctrine Migrations.

پیکربندی Doctrine ORM
-----------------------------

.. index::
    single: Doctrine;Configuration

Doctrine چگونه نحوه‌ی اتصال به پایگاه‌داده را می‌داند؟ recipe مربوط به Doctrine، یک فایل پیکربندی ``config/packages/doctrine.yaml`` که رفتار را کنترل می‌کند، اضافه می‌کند. در این فایل، *database DSN* تنظیم اصلی است. یک رشته (string) که شامل تمام اطلاعات مربوط به اتصال است: اعتبارنامه‌ها، میزبان (host)، درگاه (port) و غیره. به صورت پیش‌فرض، Doctrine به دنبال متغیر محیط ``DATABASE_URL`` می‌گردد.

تقریباً تمام بسته‌های نصب‌شده دارای یک پیکربندی در پوشه‌ی ``config/packages/`` هستند. بیشتر اوقات، مقادیر پیش‌فرض با دقت انتخاب شده‌اند تا برای اکثر اپلیکیشن‌ها کار کنند.

درک قراردادهای کار با متغیر محیط در سیمفونی
-----------------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

شما می‌توانید ``DATABASE_URL`` را به صورت دستی در فایل ``.env`` یا ``.env.local`` تعریف کنید. در حقیقت به لطف recipe مربوط به بسته، شما یک مثال ``DATABASE_URL`` در فایل ``.env`` ‌تان می‌بینید. اما چون درگاه محلی PostgreSQL که توسط Docker ارائه شده است، می‌تواند تغییر کند، این روش بسیار مشقت‌بار خواهد بود. یک راه بهتر وجود دارد.

به جای هاردکد کردن ``DATABASE_URL`` درون فایل، می‌توانیم برای تمام فرامین یک پیشوند ``symfony`` قرار دهیم. به این طریق، تمام سرویس‌های اجرا‌شده توسط Docker و/یا Upsun (زمانی که تونل باز است) شناسایی شده و متغیرهای محیط به صورت خودکار تنظیم می‌گردند.

به لطف این متغیرهای محیط، Docker Compose و Upsun به صورت یکپارچه با هم کار می‌کنند.

.. index::
    single: Symfony CLI;var:export

با اجرای ``symfony var:export``، تمام متغیرهای محیط ارائه‌شده را بررسی کنید:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

*نامِ سرویس* ``database`` را که در پیکربندی Docker و Upsun موجود بود، به خاطر می‌آورید؟ این نامِ سرویس به عنوان پیشوند برای تعریف متغیرهای محیط، همچون ``DATABASE_URL`` استفاده می‌شود. اگر سرویس شما مطابق با قراردادهای سیمفونی نامگذاری شده باشد، هیچ پیکربندی دیگری لازم نیست.

.. note::

    پایگاه‌داده‌ها تنها خدماتی نیستند که از قراردادهای سیمفونی بهره می‌برند. برای نمونه، همین رویه برای Mailer هم برقرار است (از طریق متغیر محیط ``MAILER_DSN``).

تغییر مقدار پیش‌فرض DATABASE_URL در فایل .env
------------------------------------------------------------------

ما هنوز فایل ``.env`` را جهت تنظیم مقدار پیش‌فرض ``DATABASE_URL`` برای استفاده از PostgreSQL تغییر می‌دهیم:

.. code-block:: diff

    --- i/.env
    +++ w/.env
    @@ -26,7 +26,7 @@ APP_SECRET=ce2ae8138936039d22afb20f4596fe97
     # DATABASE_URL="sqlite:///%kernel.project_dir%/var/data_%kernel.environment%.db"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=8.0.32&charset=utf8mb4"
     # DATABASE_URL="mysql://app:!ChangeMe!@127.0.0.1:3306/app?serverVersion=10.11.2-MariaDB&charset=utf8mb4"
    -DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
    +DATABASE_URL="postgresql://127.0.0.1:5432/db?serverVersion=16&charset=utf8"
     ###< doctrine/doctrine-bundle ###

     ###> symfony/messenger ###

چرا لازم است اطلاعات در دو جای مختلف تکرار شوند؟ زیرا در برخی پلتفرم‌های ابری، در *زمان ساخت (build time)*، ممکن است هنوز URL پایگاه‌داده مشخص نباشد ولی Doctrine نیاز دارد تا موتور پایگاه‌داده را بشناسد تا پیکربندی خود را ایجاد کند. بنابراین، میزبان (host)، نام کاربری و رمزعبور واقعاً اهمیت ندارند.

ایجاد کلاس‌های موجودیت
-------------------------------------------

یک کنفرانس می‌تواند با تعدادی ویژگی توصیف شود:

* شهری (*city*) که کنفرانس در آن برگزار شده است؛

* سالی (*year*) که کنفرانس در آن برگزار شده است؛

* یک پرچم بین‌المللی (*international*) که مشخص می‌کند آیا کنفرانس محلی است یا بین‌المللی (SymfonyLive در مقابل SymfonyCon).

.. index:: ! Command;make:entity

باندل Maker می‌تواند برای تولید یک کلاس ( یک کلاسِ *Entity*) که یک کنفرانس را نمایندگی می‌کند، به ما کمک نماید.

اکنون زمان تولید موجودیت ``Conference`` فرا رسیده است:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

این فرمان تعاملی است: این فرمان ما را در فرآیند افزودن تمام فیلدهایی که نیاز داریم، راهنمایی می‌کند. از این پاسخ‌ها استفاده کنید (اکثر آن‌ها پیش‌فرض هستند، بنابراین می‌تواند کلید «Enter» را برای استفاده از آن‌ها، بفشارید):

* ``city``، ``string``، ``255``، ``no``؛
* ``year``، ``string``، ``4``، ``no``؛
* ``isInternational``، ``boolean``، ``no``.

هنگامی که فرمان را اجرا می‌کنید، این خروجی کامل است:

.. code-block:: text
    :emphasize-lines: 8,11,14,17,22,25,28,31,36,39,42,45
    :class: ignore

     created: src/Entity/Conference.php
     created: src/Repository/ConferenceRepository.php

     Entity generated! Now let's add some fields!
     You can always add more fields later manually or by re-running this command.

     New property name (press <return> to stop adding fields):
     > city

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > year

     Field type (enter ? to see all types) [string]:
     >

     Field length [255]:
     > 4

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     > isInternational

     Field type (enter ? to see all types) [boolean]:
     >

     Can this field be null in the database (nullable) (yes/no) [no]:
     >

     updated: src/Entity/Conference.php

     Add another property? Enter the property name (or press <return> to stop adding fields):
     >



      Success!


     Next: When you're ready, create a migration with make:migration

کلاس ``Conference`` در فضایِ نام ``App\Entity\`` ذخیره شده است.

همچنین این فرمان، یک کلاس Doctrine *repository* تولید کرده است: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

کدِ تولید‌شده، چیزی شبیه به این است (تنها بخش کوچکی از فایل در اینجا تکرار شده است):

.. code-block:: php
    :caption: src/Entity/Conference.php
    :class: ignore

    namespace App\Entity;

    use App\Repository\ConferenceRepository;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: ConferenceRepository::class)]
    class Conference
    {
        #[ORM\Id]
        #[ORM\GeneratedValue]
        #[ORM\Column]
        private ?int $id = null;

        #[ORM\Column(length: 255)]
        private ?string $city = null;

        // ...

        public function getCity(): ?string
        {
            return $this->city;
        }

        public function setCity(string $city): static
        {
            $this->city = $city;

            return $this;
        }

        // ...
    }

توجه کنید که این کلاس به تنهایی، یک کلاس ساده‌ی PHP است و اثری از Doctrine وجود ندارد. از attributeها برای افزودن فراداده‌های لازم برای Doctrine جهت تصویر کردن کلاس به جدول پایگاه‌داده‌ی مربوطه، استفاده شده است.

Doctrine یک ویژگی ``id`` را برای ذخیره کردن کلید اصلی ردیف در جدول پایگاه‌داده، اضافه کرده است. این کلید (``#[ORM\Id]``) به صورت خودکار (``#[ORM\GeneratedValue]``) و از طریق یک استراتژی که بستگی به موتور پایگاه‌داده دارد، تولید می‌شود.

.. index::
    single: Command;make:entity

حالا، یک کلاسِ موجودیت برای کامنت‌های کنفرانس تولید کنید:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

این پاسخ‌ها را وارد کنید:

* ``author``، ``string``، ``255``، ``no``؛
* ``text``، ``text``، ``no``؛
* ``email``، ``string``، ``255``، ``no``؛
* ``createdAt``، ``datetime_immutable``، ``no``.

متصل‌کردن موجودیت‌ها
-----------------------------------------

.. index::
    single: Command;make:entity

دو موجودیت Conference و Comment، باید به یکدیگر متصل شوند. یک Conference می‌تواند صفر یا چندین Comment داشته باشد که به آن رابطه‌ی یک به چند (*one-to-many*) می‌گویند.

برای افزودن این رابطه به کلاس ``Conference``، مجدداً از فرمان ``make:entity`` استفاده کنید:

.. code-block:: terminal
    :class: answers(comments||OneToMany||Comment||conference||no||yes)

    $ symfony console make:entity Conference

.. code-block:: text
    :emphasize-lines: 4,7,10,15,18,27
    :class: ignore

     Your entity already exists! So let's add some new fields!

     New property name (press <return> to stop adding fields):
     > comments

     Field type (enter ? to see all types) [string]:
     > OneToMany

     What class should this entity be related to?:
     > Comment

     A new property will also be added to the Comment class...

     New field name inside Comment [conference]:
     >

     Is the Comment.conference property allowed to be null (nullable)? (yes/no) [yes]:
     > no

     Do you want to activate orphanRemoval on your relationship?
     A Comment is "orphaned" when it is removed from its related Conference.
     e.g. $conference->removeComment($comment)

     NOTE: If a Comment may *change* from one Conference to another, answer "no".

     Do you want to automatically delete orphaned App\Entity\Comment objects (orphanRemoval)? (yes/no) [no]:
     > yes

     updated: src/Entity/Conference.php
     updated: src/Entity/Comment.php

.. note::

    اگر در پاسخ به نوع (type)، مقدار ``?`` را وارد کنید، تمام انواع پشتیبانی‌شده را دریافت خواهید کرد:

    .. code-block:: text
        :class: ignore

        Main types
          * string
          * text
          * boolean
          * integer (or smallint, bigint)
          * float

        Relationships / Associations
          * relation (a wizard will help you build the relation)
          * ManyToOne
          * OneToMany
          * ManyToMany
          * OneToOne

        Array/Object Types
          * array (or simple_array)
          * json
          * object
          * binary
          * blob

        Date/Time Types
          * datetime (or datetime_immutable)
          * datetimetz (or datetimetz_immutable)
          * date (or date_immutable)
          * time (or time_immutable)
          * dateinterval

        Other Types
          * decimal
          * guid
          * json_array

.. index::
    single: Attributes;ORM\\ManyToOne
    single: Attributes;ORM\\JoinColumn
    single: Attributes;ORM\\OneToMany

پس از افزودن رابطه، به diff کامل کلاس‌های موجودیت نگاهی بیندازید:

.. code-block:: diff
    :class: ignore

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -23,6 +23,10 @@ class Comment
         #[ORM\Column]
         private ?\DateTimeImmutable $createdAt = null;

    +    #[ORM\ManyToOne(inversedBy: 'comments')]
    +    #[ORM\JoinColumn(nullable: false)]
    +    private ?Conference $conference = null;
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -88,4 +92,16 @@ class Comment

             return $this;
         }
    +
    +    public function getConference(): ?Conference
    +    {
    +        return $this->conference;
    +    }
    +
    +    public function setConference(?Conference $conference): static
    +    {
    +        $this->conference = $conference;
    +
    +        return $this;
    +    }
     }
    --- i/src/Entity/Conference.php
    +++ w/src/Entity/Conference.php
    @@ -2,6 +2,8 @@

     namespace App\Entity;

    +use Doctrine\Common\Collections\ArrayCollection;
    +use Doctrine\Common\Collections\Collection;
     use Doctrine\ORM\Mapping as ORM;

     /**
    @@ -20,6 +22,19 @@ class Conference
         #[ORM\Column]
         private ?bool $isInternational = null;

    +    /**
    +     * @var Collection<int, Comment>
    +     */
    +    #[ORM\OneToMany(targetEntity: Comment::class, mappedBy: 'conference', orphanRemoval: true)]
    +    private Collection $comments;
    +
    +    public function __construct()
    +    {
    +        $this->comments = new ArrayCollection();
    +    }
    +
         public function getId(): ?int
         {
             return $this->id;
    @@ -71,4 +83,35 @@ class Conference

             return $this;
         }
    +
    +    /**
    +     * @return Collection<int, Comment>
    +     */
    +    public function getComments(): Collection
    +    {
    +        return $this->comments;
    +    }
    +
    +    public function addComment(Comment $comment): static
    +    {
    +        if (!$this->comments->contains($comment)) {
    +            $this->comments->add($comment);
    +            $comment->setConference($this);
    +        }
    +
    +        return $this;
    +    }
    +
    +    public function removeComment(Comment $comment): static
    +    {
    +        if ($this->comments->removeElement($comment)) {
    +            // set the owning side to null (unless already changed)
    +            if ($comment->getConference() === $this) {
    +                $comment->setConference(null);
    +            }
    +        }
    +
    +        return $this;
    +    }
     }

تمام چیزهایی که برای مدیریت رابطه به آن احتیاج دارید، برای شما تولید شده است. وقتی که کد تولید شد، از آن به بعد کد متعلق به شماست؛ می‌توانید با آزادی کامل، به هر نحوی که می‌خواهید آن را شخصی‌سازی کنید.

افزودن ویژگی‌های بیشتر
-------------------------------------------

.. index::
    single: Command;make:entity

همین الان متوجه شدم که ما اضافه‌کردن یک ویژگی برای موجودیت کامنت را فراموش کرده‌ایم: شرکت‌کنندگان ممکن است بخواهند برای نشان‌دادن بازخوردشان، یک عکس از کنفرانس را ضمیمه کنند.

``make:entity`` را یکبار دیگر اجرا کنید تا ویژگی/ستون ``photoFilename`` را از نوع ``string`` اضافه کنیم، اما اجازه دهید تا مقدار ``null`` را بپذیرد زیرا که بارگذاری عکس اختیاری است.

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

Migrateکردن پایگاه‌داده
---------------------------------------

.. index:: ! Command;make:migration

در حال حاضر مدل پروژه به صورت کامل توسط دو کلاس تولید‌شده، توصیف شده است.

در ادامه نیاز داریم تا جداول پایگاه‌داده مرتبط با این موجودیت‌های PHP را ایجاد کنیم.

*Doctrine Migrations* برای انجام این کار، عالی است. این ابزار هم‌اکنون به عنوان بخشی از وابستگی ``orm``، نصب شده است.

یک *migration*، یک کلاس است که تغییرات لازم برای بروزرسانی شمای پایگاه‌داده را از وضعیت فعلی به وضعیت جدید که توسط attributeها تعریف شده است، توصیف می‌کند. از آنجایی که پایگاه‌داده در حال  حاضر خالی است، migration باید شامل ۲ ایجاد جدول باشد.

بیایید ببینیم که Doctrine چه چیزی تولید می‌کند:

.. code-block:: terminal

    $ symfony console make:migration

به نام فایل تولیدشده در خروجی توجه کنید (نامی شبیه ``migrations/Version20191019083640.php``):

.. code-block:: php
    :caption: migrations/Version20191019083640.php
    :class: ignore

    namespace DoctrineMigrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version00000000000000 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            // this up() migration is auto-generated, please modify it to your needs
            $this->addSql('CREATE SEQUENCE comment_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE SEQUENCE conference_id_seq INCREMENT BY 1 MINVALUE 1 START 1');
            $this->addSql('CREATE TABLE comment (id INT NOT NULL, conference_id INT NOT NULL, author VARCHAR(255) NOT NULL, text TEXT NOT NULL, email VARCHAR(255) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, photo_filename VARCHAR(255) DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX IDX_9474526C604B8382 ON comment (conference_id)');
            $this->addSql('CREATE TABLE conference (id INT NOT NULL, city VARCHAR(255) NOT NULL, year VARCHAR(4) NOT NULL, is_international BOOLEAN NOT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE comment ADD CONSTRAINT FK_9474526C604B8382 FOREIGN KEY (conference_id) REFERENCES conference (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }

        public function down(Schema $schema): void
        {
            // ...
        }
    }

به‌روزرسانی پایگاه‌داده‌ی محلی
-------------------------------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

حالا می‌توانید migration تولیدشده را اجرا  کنید تا شمای پایگاه‌داده‌ی محلی به‌روزرسانی شود:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

حالا شمای پایگاه‌داده‌ی محلی به‌روز و آماده‌ی ذخیره‌ی مقداری داده است.

به‌روزرسانی پایگاه‌داده‌ی عمل‌آوری
----------------------------------------------------------------------

گام‌های لازم برای migrateکردن پایگاه‌داده‌ی عمل‌آوری، دقیقاً همان‌هایی است که تا الان با آن آشنا شده‌اید: تغییرات را commit کرده و مستقر کنید.

هنگامی که پروژه را مستقر می‌کنید، Upsun کد را به‌روز می‌کند اما علاوه بر آن، در صورت وجود، migration پایگاه‌داده را نیز اجرا می‌کند (اگر فرمان ``doctrine:migrations:migrate`` وجود داشته باشد، تشخیص می‌دهد).

.. sidebar:: بیشتر بدانید

    * `پایگاه‌داده‌ها و Doctrine ORM`_ در اپلیکیشن‌های سیمفونی؛

    * `آموزش تصویری Doctrine در SymfonyCasts`_؛

    * `کار با Doctrine Associations/Relations`_؛

    * `DoctrineMigrationsBundle docs`_.

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`پایگاه‌داده‌ها و Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`آموزش تصویری Doctrine در SymfonyCasts`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`کار با Doctrine Associations/Relations`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle docs`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
