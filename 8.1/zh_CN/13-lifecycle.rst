管理 Doctrine 对象的生命周期
=====================================

当新增一条评论时，``createdAt`` 这个字段最好可以自动设为当前的日期和时间。

Doctrine 在对象的生命周期中（在新行被插入到数据库之前、在对应行被更新之后......），有不同的方式来操作对象以及对象的属性。

定义生命周期的回调方法
---------------------------------

.. index::
    single: Doctrine;Lifecycle
    single: Attributes;ORM\\Entity
    single: Attributes;ORM\\HasLifecycleCallbacks
    single: Attributes;ORM\\PrePersist

当要执行的行为无需任何服务对象，而且只是针对一种实体类时，可以在该实体类里定义一个回调方法：

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

对象的数据被第一次存储在数据库时会触发 ``ORM\PrePersist`` *事件*。当触发时，``setCreatedAtValue()`` 方法会被调用，当前的日期和时间就会用来设置 ``createdAt`` 属性的值。

为会议添加 *slug*
----------------------

会议页面的 URL 对会议缺乏描述性：``/conference/1``。更重要的是，这些 URL 依赖于一个实现细节（数据库表的主键被泄露了）。

用类似 ``/conference/paris-2020`` 这样的 URL 来代替怎么样？那看上去好得多。``paris-2020`` 就是我们所谓的会议 *slug*。

.. index::
    single: Command;make:entity

为会议增加一个 ``slug`` 属性（一个不能取值 null 的 255 长度的字符串）：

.. code-block:: terminal
    :class: answers(slug||string||255||no)

    $ symfony console make:entity Conference

.. index::
    single: Command;make:migration

添加一个数据库结构迁移文件，用来增加一个新的列：

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

执行这个新的结构迁移：

.. code-block:: terminal
    :class: ignore

    $ symfony console doctrine:migrations:migrate

得到一个错误？这是预料之中的。为什么呢？因为我们要求这个 slug 不能是 null 值，但是当结构迁移执行时，数据库中已有的会议行会为 slug 设置一个 ``null`` 值。让我们通过调整一下该迁移文件来修复这个错误。

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

这里的技巧是先增加这个 slug 列并且允许它取值为 ``null``，然后再把行里的 slug 设置成一个非 ``null`` 值，最后把这个列设置回不能取 ``null`` 值。

.. note::

    对于真实项目，在 SQL 中使用 ``CONCAT(LOWER(city), '-', year)`` 可能还不足以解决问题。在这种情况下，我们需要一个 *真正* 的 Slugger。

.. index::
    single: Command;doctrine:migrations:migrate

数据库结构迁移现在应该可以顺利执行了：

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

.. index::
    single: Attributes;ORM\\UniqueEntity
    single: Attributes;ORM\\Column
    single: Components;Validator

因为应用程序马上会使用 slug 来查找每个会议，我们来调整下会议的实体类，确保 slug 的值在数据库中是唯一的：

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

正如你所料，我们需要执行一次数据库结构迁移：

.. code-block:: terminal

    $ symfony console make:migration

.. index::
    single: Command;doctrine:migrations:migrate

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

生成 slug
-----------

.. index::
    single: Components;String
    single: Slug

生成一个可读性良好的 slug，并将它用于 URL（任何非 ASCII 字符都会被编码），这是个有挑战的工作，尤其是对于英语之外的语言。比如你该如何把 ``é`` 转换成 ``e`` 呢？

不必重新发明轮子，让我们来用 Symfony 的 ``String`` 组件，它使字符串操作变得很容易，而且它也提供了一个 *slugger*。

在 ``Conference`` 类里增加一个 ``computeSlug()`` 方法，它会根据会议数据计算出 slug 的值：

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

只有当前 slug 为空或者是设为了 ``-`` 这个特殊值的时候，``computeSlug()`` 方法才会计算 slug 值。为什么我们要用 ``-`` 这个特殊值呢？因为在后台新增一个会议时，slug 是必填的。所以我们需要一个非空值，它告诉应用程序，我们要自动生成 slug。

定义一个复杂的生命周期回调方法
---------------------------------------------

.. index::
    single: Doctrine;Entity Listener

和 ``createdAt`` 一样，``slug`` 应该被自动设置。当会议被更新时，``computeSlug()`` 方法要被自动调用。

但是这个方法依赖于 ``SluggerInterface`` 接口的一个实现，所以我们无法像之前那样增加一个 ``prePersist`` 事件（我们无法注入 slugger）。

转而创建一个 Doctrine 里针对实体的监听器：

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

注意到当新建或更新一个会议时，slug 也会更新（新建时调用 ``prePersist()`` 方法，更新时调用 ``preUpdate()`` 方法）。

在服务容器中配置服务
------------------------------

.. index::
    single: Components;Dependency Injection
    single: Dependency Injection

到目前为止，我们还没有谈到 Symfony 中一个重要的组成部分，就是 *依赖注入容器*，它负责管理各个 *服务*：在需要的时候创建和注入服务。

一个 *服务* 是一个提供某项功能的“全局”对象（比如一个 *mailer* 、一个 *logger* 、一个 *slugger* 等等），而不是一个 *数据对象* （比如 Doctrine 实体的实例）。

你极少需要去直接操作容器，因为在你需要服务的时候，它会自动注入服务对象：例如，当你在控制器方法里对参数进行类型提示时，容器会注入对应类型的服务对象。

如果在前面的步骤里，你想要知道事件的监听器是如何注册的，那么现在你有答案了：那就是容器。当某个类实现了一个特定的接口，容器就会知道应该以某种方式注册这个类。

在这里，因为我们的类没有实现任何接口，也没有继承任何基类，Symfony 不知道如何自动配置它。我们可以改用一个属性来告诉 Symfony 容器如何装配它：

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

    不要把 Doctrine 的事件监听器和 Symfony 的事件监听器混淆起来。尽管它们看上去很相似，但它们用的底层代码架构是不一样的。

在应用中使用 slug
-----------------------

试着在后台里添加更多会议，并且更新一个现有会议的城市或年份；除非你使用特殊的 ``-`` 值，否则 slug 不会更新。

.. index::
    single: Twig;for
    single: Twig;if
    single: Twig;path
    single: Attributes;Route

最后一个改动，就是更新控制器和模板，让它们在路由中使用会议的 ``slug`` 来代替 ``id``。由于路由参数不再是实体的主键，使用 ``{slug:conference}`` 语法来告诉 Symfony 通过匹配 ``slug`` 属性来获取 ``$conference``；不再需要 ``#[MapEntity]`` 属性了：

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

现在会议页面应该可以使用它的 slug 来打开：

.. figure:: screenshots/slug.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. sidebar:: 深入学习

    * `Doctrine 的事件系统`_ （生命周期回调和监听器，实体的监听器和生命周期订阅器）；

    * `String 组件文档`_；

    * `服务容器`_；

    * `Symfony 服务速查表`_。

.. _`Doctrine 的事件系统`: https://symfony.com/doc/current/doctrine/events.html
.. _`String 组件文档`: https://symfony.com/doc/current/components/string.html
.. _`服务容器`: https://symfony.com/doc/current/service_container.html
.. _`Symfony 服务速查表`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/services_en_42.pdf
