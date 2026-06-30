设置管理后台
==================

.. index::
    single: EasyAdmin
    single: Admin
    single: Backend

把将要举办的会议录入到数据库是项目管理员的工作。所谓的 *管理后台* 是网站中一个受保护的区域，用来让 *项目管理员* 管理网站数据，处理提交的反馈和做其它一些事。

我们如何快速做一个后台呢？通过用一个 bundle，它可以根据项目的数据模型来生成后台！EasyAdmin 最适合不过了。

安装更多依赖
----------------

即便 ``webapp`` 包已经自动添加了很多很好的包，但对于一些更具体的功能，我们还需要添加更多依赖。如何添加更多依赖呢？通过 Composer。除了“常规的” Composer 包，我们还会用到两种“特殊”的包：

* *Symfony 组件（Symfony Components）*：实现核心功能和大多数应用都需要的底层抽象的包（路由、控制台、HTTP 客户端、邮件、缓存......）；

* *Symfony 扩展包（Symfony Bundles）*：添加高层功能或提供与第三方库集成的包（扩展包大多由社区贡献）。

我们来把 EasyAdmin 加进项目的依赖中：

.. code-block:: terminal

    $ symfony composer req "easycorp/easyadmin-bundle:^5"

*别名（Aliases）* 不是 Composer 的特性，而是 Symfony 为了让你更轻松而提供的一个概念。别名是流行 Composer 包的快捷方式。想为你的应用添加 ORM？require ``orm`` 就行。想开发一个 API？require ``api``。这些别名会自动解析为一个或多个常规的 Composer 包。它们是 Symfony 核心团队做出的有主见的选择。

另一个不错的特性是你总是可以省略 ``symfony`` 这个 vendor 名。用 ``cache`` 来代替 ``symfony/cache``。

.. tip::

    还记得我们之前提到过的名为 ``symfony/flex`` 的 Composer 插件吗？别名就是它的特性之一。

配置 EasyAdmin
----------------

EasyAdmin 会基于一些特定的控制器，来为你的应用程序自动生成一个管理后台。

作为使用 EasyAdmin 的第一步，我们来创建一个“网站管理仪表盘”，它将是管理网站数据的主入口。

.. code-block:: terminal
    :class: answers(DashboardController||src/Controller/Admin/)

    $ symfony console make:admin:dashboard

接受默认的答案，这会创建如下的控制器：

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

按照约定，所有的管理控制器都放在它们自己的 ``App\Controller\Admin`` 命名空间下。

用 ``/admin`` 路径访问这个生成的后台，该路径是由 ``#[AdminDashboard]`` 属性来配置的；你可以把这个 URL 改成任何你想要的形式：

.. figure:: screenshots/easy-admin-empty.png
    :alt: /admin
    :align: center
    :figclass: with-browser

哇！我们有了一个很好看的管理界面外壳，可以根据我们的需要对它进行定制。

.. index::
    single: CRUD

接下去的步骤是创建控制器来管理会议和评论。

在仪表盘的控制器里，你可能已经注意到了 ``configureMenuItems()`` 方法，它带有一条有关增加链接到 “CRUD” 类的注释。**CRUD** 是英文中“增、查、改、删”的首字母缩略词，它们是你想要对任何实体做的 4 个基本操作。这也正是我们希望管理后台为我们去做的；EasyAdmin 甚至更进一步，为我们处理了搜索和过滤。

我们来为会议生成一个 CRUD 类：

.. code-block:: terminal
    :class: answers(1||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

选择 ``1`` 来为会议创建一个管理界面，其余的问题都选择默认答案。这会生成如下文件：

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

对评论也一样处理：

.. code-block:: terminal
    :class: answers(0||src/Controller/Admin/||App\\Controller\\Admin)

    $ symfony console make:admin:crud

最后一步是把会议和评论的 CRUD 管理类链接到仪表盘：

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

我们改写了 ``configureMenuItems()`` 方法，加了配有相关图标的会议和评论菜单项，也加了一个回到网站首页的链接。``ConferenceCrudController`` 和 ``CommentCrudController`` 类和仪表盘位于同一个 ``App\Controller\Admin`` 命名空间下，所以它们不需要额外的 ``use`` 语句。

EasyAdmin 通过 ``MenuItem::linkTo()`` 方法来暴露出一个 API 接口，便于链接到实体对应的 CRUD 类，该方法接收 CRUD 控制器类作为参数。

主仪表盘页面目前还是空的。该页面可以用来展示一些统计信息，或是任何相关的信息。由于我们没有什么重要的内容要展示，我们把它重定向到会议列表页：

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

当展示实体间的关系时（和评论关联的一场会议），EasyAdmin 会尝试用会议实体的字符串表示。默认情况下，如果实体没有定义 ``__toString()`` 这个“魔术”方法，它的惯例就是使用实体名和主键（比如 ``Conference #1``）。为了使展示的内容更具意义，我们来为 ``Conference`` 类定义这个方法：

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

现在你可以在管理后台里直接添加/修改/删除会议。你可以玩玩看，并且添加至少一个会议。

.. figure:: screenshots/easy-admin.png
    :alt: /admin
    :align: center
    :figclass: with-browser

定制 EasyAdmin
----------------

默认的后台运行得很好，但是我们可以在许多方面对它进行定制，以提高使用体验。我们来对 Comment 实体类做一些简单的改动，以此演示一些可能的定制方式：

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

为了定制 ``Comment`` 这部分，要在 ``configureFields()`` 方法中明确列出字段，这让我们能以想要的次序来对它们进行排列。有些字段要更多配置下，比如在列表页隐藏文本字段。

添加几个不带照片的评论。现在先手工设置下日期；在后面的步骤中，我们会让 ``createdAt`` 列自动填充。

.. figure:: screenshots/easy-admin-comments.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

``configureFilters()`` 方法定义了在搜索之外还要暴露哪些过滤器。

.. figure:: screenshots/easy-admin-filter.png
    :alt: /admin?crudAction=index&crudId=2bfa220&menuIndex=2&submenuIndex=-1
    :align: center
    :figclass: with-browser

这些改动只是对于 EasyAdmin 定制可能性的一个小小介绍。

在后台里玩玩，比如可以用会议来过滤评论，或者根据邮件来搜索评论。目前唯一的问题是任何人都可以进入后台。别担心，我们会在以后的步骤中将它保护起来。

.. code-block:: terminal
    :class: hide

    $ symfony console dbal:run-sql "TRUNCATE conference RESTART IDENTITY CASCADE"

.. sidebar:: 深入学习

    * `EasyAdmin 文档`_；

    * `Symfony 框架的配置参考`_；

    * `PHP 魔术方法`_。

.. _`EasyAdmin 文档`: https://symfony.com/bundles/EasyAdminBundle/4.x/index.html
.. _`Symfony 框架的配置参考`: https://symfony.com/doc/current/reference/configuration/framework.html
.. _`PHP 魔术方法`: https://www.php.net/manual/en/language.oop5.magic.php
