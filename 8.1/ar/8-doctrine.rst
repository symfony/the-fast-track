وصف هيكل البيانات
================================

.. index::
    single: Doctrine
    single: Database

للتعامل مع قاعدة البيانات من PHP ، سوف نعتمد على `Doctrine`_ ، وهي مجموعة من المكتبات التي تساعد المطورين على إدارة قواعد البيانات: Doctrine DBAL (طبقة تجريد قاعدة البيانات) ، Doctrine ORM (مكتبة لمعالجة محتوى قاعدة البيانات الخاصة بنا باستخدام كائنات PHP) ، و Doctrine Migrations.

تكوين Doctrine ORM
-----------------------

.. index::
    single: Doctrine;Configuration

كيف يقوم Doctrine بمعرفة الاتصال بقاعدة البيانات؟ وصفة Doctrine تقوم بإضافة ملف تكوين، ``config/packages/doctrine.yaml`` الذي بدوره يقوم بالتحكم ب Doctrine سلوكيا. الإعداد الرئيسي هنا هو *Database DSN*، نص يحوي كل المعلومات عن الاتصال، بيانات التوثيق، المضيف، المنفذ .. الخ، افتراضيا، تبحث Doctrine عن متغير البيئة ``DATABASE_URL``.

تحتوي جميع الحزم المثبتة تقريبًا على تكوين ضمن دليل ``config/packages/``. في معظم الأوقات ، تم اختيار الإعدادات الافتراضية بعناية لتعمل مع معظم التطبيقات.

فهم اتفاقيات متغير البيئة الخاص ب Symfony
---------------------------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

يمكنك تحديد ``DATABASE_URL`` يدويًا في ملف ``.env`` أو ``.env.local``. في الحقيقة، بفضل وصفة الحزمة ، سترى ``DATABASE_URL`` مثالًا  في ملف ``.env``. ولكن نظرًا لأن المنفذ المحلي لـ PostgreSQL الذي تتعرض له Docker يمكن أن يتغير، إنه أمر مرهق للغاية. هناك طريقة افضل.

بدلاً من الترميز الثابت ``DATABASE_URL`` في ملف ، يمكننا بداية جميع الأوامر بـ ``symfony``. سيؤدي هذا إلى اكتشاف الخدمات التي يديرها Docker و / أو Upsun (عندما يكون النفق مفتوحًا) وتعيين متغير البيئة تلقائيًا.

يعمل Docker Compose و Upsun بسلاسة مع Symfony بفضل متغيرات البيئة هذه.

.. index::
    single: Symfony CLI;var:export

تحقق من جميع متغيرات البيئة المكشوفة عن طريق تنفيذ ``symfony var:export``:

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

هل تتذكر``database`` *اسم الخدمة* المستخدمة في تكوينات Docker و Upsun؟ يتم استخدام أسماء الخدمات كبادئات لتحديد متغيرات البيئة مثل ``DATABASE_URL``. إذا تم تسمية خدماتك وفقًا لاتفاقيات Symfony ، فلن تكون هناك حاجة إلى تكوين آخر.

.. note::

    قواعد البيانات ليست هي الخدمة الوحيدة التي تستفيد من اتفاقيات Symfony. ينطبق الشيء نفسه على Mailer ، على سبيل المثال (عبر متغير البيئة ``MAILER_DSN``).

تغيير القيمة الافتراضية DATABASE_URL في .env
-------------------------------------------------------------------

سنستمر في تغيير ملف ``.env`` لإعداد  ``DATABASE_URL`` لاستخدام PostgreSQL:

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

لماذا تحتاج المعلومات إلى تكرارها في مكانين مختلفين؟ لأنه في بعض الأنظمة الأساسية السحابية ، في *وقت البناء* ، قد لا يكون عنوان URL لقاعدة البيانات معروفًا بعد ، لكن Doctrine بحاجة إلى معرفة محرك قاعدة البيانات لبناء تكوينها. لذلك ، المضيف ، اسم المستخدم وكلمة المرور لا يهم حقا.

إنشاء فئات الكيان Entity Classes
-----------------------------------------------

يمكن وصف المؤتمر ببعض الخصائص:

* *المدينة* حيث يتم تنظيم المؤتمر؛

* *السنة* متى تم انعقاد المؤتمر؛

* علامة *دولية* للإشارة إلى ما إذا كان المؤتمر محليًا أم دوليًا (SymfonyLive vs SymfonyCon).

.. index:: ! Command;make:entity

يمكن أن تساعدنا حزمة Maker في إنشاء فئة (فئة *Entity*) تمثل مؤتمرًا.

حان الوقت الآن لتوليد كيان ``Conference``:

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

هذا الأمر تفاعلي: سوف يرشدك خلال عملية إضافة جميع الحقول التي تحتاج إليها. استخدم الإجابات التالية (معظمها هي الإعدادات الافتراضية ، بحيث يمكنك الضغط على مفتاح "Enter" لاستخدامها):

* ``city``, ``string``, ``255``, ``no``؛
* ``year``, ``string``, ``4``, ``no``؛
* ``isInternational``, ``boolean``, ``no``.

هذا هو الإخراج الكامل عند تشغيل الأمر:

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

تم تخزين فئة "المؤتمر" ضمن مساحة الاسم ``App\Entity\``.

قام الأمر أيضًا بإنشاء فئة Doctrine *repository*: ``App\Repository\ConferenceRepository``.

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

يبدو الرمز الذي تم إنشاؤه كما يلي (يتم نسخ جزء صغير فقط من الملف هنا):

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

لاحظ أن الفئة class نفسه عبارة عن فئة PHP عادي بدون أي علامات على Doctrine. يتم استخدام السمات Attributes لإضافة بيانات تعريف مفيدة لـ Doctrine لتعيين الفئة إلى جدول قاعدة البيانات ذي الصلة.

أضافت Doctrine خاصية "id" لتخزين المفتاح الأساسي للصف في جدول قاعدة البيانات. يتم إنشاء هذا المفتاح (``#[ORM\Id]``) تلقائيًا (``#[ORM\GeneratedValue]``) عبر استراتيجية تعتمد على مشغل قاعدة البيانات.

.. index::
    single: Command;make:entity

الآن ، قم بإنشاء فئة كيان لتعليقات المؤتمر:

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

أدخل الإجابات التالية:

* ``author``, ``string``, ``255``, ``no``؛
* ``text``, ``text``, ``no``؛
* ``email``, ``string``, ``255``, ``no``؛
* ``createdAt``, ``datetime``, ``no``.

ربط الكيانات
-----------------------

.. index::
    single: Command;make:entity

ينبغي ربط الكيانين ، المؤتمر والتعليق ، معًا. يمكن أن يحتوي المؤتمر على صفر أو أكثر من التعليقات ، والتي تسمى علاقة *واحد لكثير*.

استخدم الأمر ``make:entity`` مرة أخرى لإضافة هذه العلاقة إلى فئة ``Conference``:

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

    إذا أدخلت ``?`` كإجابة للنوع ، فستحصل على جميع الأنواع المدعومة:

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

ألقِ نظرة على الفرق الكامل لفئات الكيانات بعد إضافة العلاقة:

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

تم إنشاء كل ما تحتاجه لإدارة العلاقة من أجلك. وقت أن يتم توليد الرمز، يصبح ملكا لك. لا تتردد في تخصيصه بالطريقة التي تريدها.

إضافة المزيد من الخصائص
-------------------------------------------

.. index::
    single: Command;make:entity

لقد أدركت تمامًا أننا نسينا إضافة خاصية واحدة على كيان التعليق: قد يرغب الحاضرون في إرفاق صورة للمؤتمر لتوضيح ملاحظاتهم.

شغِّل ``make:entity``  مرة أخرى وأضف خاصية / عمود ``photoFilename`` من نوع ``string`` ، لكن اسمح له أن يكون ``null`` لأن تحميل الصورة اختياري:

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

ترحيل قاعدة البيانات
--------------------------------------

.. index:: ! Command;make:migration

يتم الآن وصف نموذج المشروع بالكامل من خلال فئتين تم إنشاؤهما.

بعد ذلك ، نحتاج إلى إنشاء جداول قاعدة البيانات المتعلقة بكيانات PHP هذه.

*Doctrine Migrations* هي الأداة المثالية لمثل هذه المهمة. تم تثبيتها بالفعل كجزء من تبعية ``orm``.

*الترحيل* هو فئة تصف التغييرات اللازمة لتحديث مخطط قاعدة البيانات من حالته الحالية إلى الحالة الجديدة المحددة بواسطة سمات الكيان. نظرًا لأن قاعدة البيانات فارغة في الوقت الحالي ، يجب أن تتكون عملية الترحيل من إنشاء جدولين.

دعونا نرى ما أنتجته Doctrine:

.. code-block:: terminal

    $ symfony console make:migration

لاحظ اسم الملف الذي تم إنشاؤه في الإخراج (اسم يشبه ``migrations/Version20191019083640.php``):

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

تحديث قاعدة البيانات المحلية
-----------------------------------------------------

.. index:: ! Command;doctrine:migrations:migrate

يمكنك الآن تشغيل الترحيل الذي تم إنشاؤه لتحديث مخطط قاعدة البيانات المحلية:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

مخطط قاعدة البيانات المحلية الآن محدث ، جاهز لتخزين بعض البيانات.

تحديث قاعدة بيانات الإنتاج
-------------------------------------------------

الخطوات اللازمة لترحيل قاعدة بيانات الإنتاج هي نفسها التي تعرفها بالفعل: قم بإجراء التغييرات ونشرها.

عند نشر المشروع ، يقوم Upsun بتحديث الكود ، لكنه يدير أيضًا ترحيل قاعدة البيانات إن وجد (يكتشف ما إذا كان الأمر ``doctrine:migrations:migrate`` موجودًا).

.. sidebar:: الذهاب أبعد من ذلك

    * `قواعد البيانات و Doctrine ORM`_ في تطبيقات Symfony؛

    * `SymfonyCasts Doctrine درس تعليمي`_؛

    * `العمل مع علاقات ترابطية Doctrine`_؛

    * `DoctrineMigrationsBundle docs`_.

.. _`قواعد البيانات و Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`SymfonyCasts Doctrine درس تعليمي`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`العمل مع علاقات ترابطية Doctrine`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle docs`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html

.. _`Doctrine`: https://www.doctrine-project.org/
