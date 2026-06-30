描述数据结构
==================

.. index::
    single: Doctrine
    single: Database

我们要依赖 `Doctrine`_ 来让 PHP 处理数据库，它由一组类库组成，这些类库可以帮助开发者管理数据库：Doctrine DBAL（一个数据库抽象层）、Doctrine ORM（一个用 PHP 对象来管理数据库内容的库）和 Doctrine Migrations。

配置 Doctrine ORM
-------------------

.. index::
    single: Doctrine;Configuration

Doctrine 是如何知道数据库连接信息的呢？Doctrine 的 recipe 添加了 ``config/packages/doctrine.yaml`` 这个配置文件，它控制了 Doctrine 的行为方式。其中主要的设置项是 *数据库的DSN*，这是一个包含了所有连接信息的字符串：账号密码、服务器名、端口等。默认情况下，Doctrine 会找一个名为 ``DATABASE_URL`` 的环境变量。

几乎所有安装好的包都会有一个配置文件放在 ``config/packages/`` 目录下。大多数情况下，默认配置项都是精心选择的，适用于大部分应用。

理解 Symfony 的环境变量约定
------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

你可以在 ``.env`` 或 ``.env.local`` 文件中手工定义 ``DATABASE_URL`` 变量。事实上，你能在 ``.env`` 文件里看到 ``DATABASE_URL`` 变量的一个例子，它是由包的 recipe 所添加。但由于 Docker 暴露出来的 PostgreSQL 端口不是固定的，这个方案会很繁琐。其实有个更好的方案。

我们不用把 ``DATABASE_URL`` 硬编码在一个文件中，我们只要在所有命令前加上 ``symfony`` 前缀。这样的话 Docker 运行的服务会被自动检测到（当隧道打开的时候，Upsun 的服务也会被检测到），环境变量也会被自动设置好。

借助于环境变量，Docker Compose 以及 Upsun 可以和 Symfony 无缝对接。

.. index::
    single: Symfony CLI;var:export

通过执行 ``symfony var:export`` 来查看所有暴露出来的环境变量：

.. code-block:: terminal

    $ symfony var:export

.. code-block:: text
    :class: ignore

    DATABASE_URL=postgres://app:!ChangeMe!@127.0.0.1:32781/app?sslmode=disable&charset=utf8
    # ...

你还记得在 Docker 和 Upsun 里使用的 ``database`` 这个 *服务名* 吗？服务名用来作为环境变量名的前缀，比如 ``DATABASE_URL``。如果你的服务根据 Symfony 的约定来命名，那么就不需要其它的配置了。

.. note::

    数据库不是唯一从 Symfony 约定中受益的服务。比如，Mailer 是另外一个例子（通过 ``MAILER_DSN`` 环境变量）。

在 .env 文件中修改 DATABASE_URL 的默认值
--------------------------------------------------

我们仍然会修改 ``.env`` 文件来设置 ``DATABASE_URL`` 的默认值，这样才能使用 PostgreSQL：

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

为什么这些信息要在两个不同的地方重复呢？因为有些云平台上在 *构建时*，数据库的信息还没确定，而 Doctrine 却需要知道用哪个数据库引擎来构建它的配置。这样说来，服务器名、用户名和密码都不重要。

创建实体类
---------------

需要一些属性来描述一个会议：

* 举行会议所在的 *城市*；

* 会议的 *年份*；

* *国际化* 选项来标明这个会议是本地的还是国际的（SymfonyLive vs SymfonyCon）。

.. index:: ! Command;make:entity

*Maker Bundle* 能帮我们生成一个代表会议的类（即一个 *实体* 类）。

现在是时候生成 ``Conference`` 实体类了：

.. code-block:: terminal
    :class: answers(city||string||255||no||year||string||4||no||isInternational||boolean||no)

    $ symfony console make:entity Conference

这个命令是交互式的：它会引导你创建所需的全部字段。在交互模式里使用以下的回复（大部分都是默认值，所以你只要按回车键就行）：

* ``city``，``string``，``255``，``no``；
* ``year``，``string``，``4``，``no``；
* ``isInternational``，``boolean``，``no``；

这是执行这个命令后的全部输出：

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

``Conference`` 类被放在 ``App\Entity\`` 命名空间下。

这个命令也会生成一个 Doctrine 的 *repository* 类：``App\Repository\ConferenceRepository``。

.. index::
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\Id
    single: Attributes;ORM\\GeneratedValue
    single: Attributes;ORM\\Column

生成的代码像下面这样（只有一小部分被复制到了这）：

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

请注意这个类本身就是一个普通的 PHP 类，和 Doctrine 没有直接关联。Doctrine 用到的元数据是通过属性（attribute）的方式添加到类里的，从而把这个类映射到相关的数据库表。

Doctrine添加了一个``id``属性来存储数据库表中的行主键。主键（``#[ORM\Id]``）的值由（``#[ORM\GeneratedValue]``）根据具体的数据库选用一个策略生成。

.. index::
    single: Command;make:entity

现在，我们来生成一个会议评论的实体类。

.. code-block:: terminal
    :class: answers(author||string||255||no||text||text||no||email||string||255||no||createdAt||datetime_immutable||no)

    $ symfony console make:entity Comment

输入以下回复：

* ``author``，``string``，``255``，``no``；
* ``text``，``text``，``no``；
* ``email``，``string``，``255``，``no``；
* ``createdAt``，``datetime_immutable``，``no``。

将多个实体类关联起来
------------------------------

.. index::
    single: Command;make:entity

我们要把 ``Conference`` 和 ``Comment`` 的实体类关联起来。一个 ``Conference`` 可以有零个或多个 ``Comment``，这种关系被称为 *一对多*。

再次使用 ``make:entity`` 命令，通过它把这种关系添加到 ``Conference`` 类：

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

    命令行会问你所需字段的类型，当你输入 ``?`` 作为回复时，你能查看所有支持的类型：

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

加好了这个关系的字段后，查看一下实体类文件的全部文件比对：

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

管理类关系所需的所有代码都已经生成了。这些代码一旦生成就属于你了，你可以按照想要的方式去修改它们。

添加更多属性
------------------

.. index::
    single: Command;make:entity

我才意识到我们忘了在评论的实体类里添加一个属性：参会者可能会想要附带一张会议的照片来表达他们的反馈。

再次执行 ``make:entity`` 命令，这次增加一个 ``string`` 类型的 ``photoFilename`` 属性/列，但是要允许它可以取 ``null`` 值，因为上传照片是可选的：

.. code-block:: terminal
    :class: answers(photoFilename||string||255||yes)

    $ symfony console make:entity Comment

迁移数据库
---------------

.. index:: ! Command;make:migration

这两个生成的类现在完整描述了项目的数据模型。

接下去，我们需要创建与实体类对应的数据库表。

*Doctrine Migrations* 是完成这一任务的完美方案。它作为 ``orm`` 依赖包的一部分已经安装好了。

如果当前数据库的结构和实体类的属性定义的结构不同，就需要进行 *迁移* （migration）操作。*迁移* 描述了当前数据库结构需要进行的更改。因为现在数据库里没有任何表，这个 *迁移* 会包含两个表的创建。

让我们来看下 Doctrine 生成了什么：

.. code-block:: terminal

    $ symfony console make:migration

请留意输出里那个生成文件的名字（一个类似 ``migrations/Version20191019083640.php`` 的名字）：

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

更新本地数据库
---------------------

.. index:: ! Command;doctrine:migrations:migrate

现在你可以运行生成的迁移来更新本地数据库结构：

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

现在本地数据库的结构已经是最新的了，可以准备存储数据。

更新生产服务器
---------------------

迁移生产数据库结构需要的步骤和你所熟知的一样：提交代码更新后部署。

当部署项目时，Upsun 会更新代码，如果需要的话，它也会执行数据库结构迁移（它会检测 ``doctrine:migrations:migrate`` 命令是否存在）。

.. sidebar:: 深入学习

    * Symfony 应用中的 `数据库和 Doctrine ORM`_；

    * `SymfonyCasts 的 Doctrine 教程`_；

    * `Doctrine 下实体类之间的关联`_；

    * `DoctrineMigrationsBundle 文档`_。

.. _`Doctrine`: https://www.doctrine-project.org/
.. _`数据库和 Doctrine ORM`: https://symfony.com/doc/current/doctrine.html
.. _`SymfonyCasts 的 Doctrine 教程`: https://symfonycasts.com/screencast/symfony-doctrine/install
.. _`Doctrine 下实体类之间的关联`: https://symfony.com/doc/current/doctrine/associations.html
.. _`DoctrineMigrationsBundle 文档`: https://symfony.com/doc/current/bundles/DoctrineMigrationsBundle/index.html
