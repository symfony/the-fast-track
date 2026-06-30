监听事件
============

当前的布局缺少一个导航的页头，这个页头可以用来返回首页，或者从一个会议跳到下一个会议。

添加网站页头
------------------

.. index::
    single: Twig;for
    single: Twig;path

任何会展示在所有页面的内容，比如页头，都是基础布局的一部分：

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -12,6 +12,15 @@
             {% endblock %}
         </head>
         <body>
    +        <header>
    +            <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    +            <ul>
    +            {% for conference in conferences %}
    +                <li><a href="{{ path('conference', { id: conference.id }) }}">{{ conference }}</a></li>
    +            {% endfor %}
    +            </ul>
    +            <hr />
    +        </header>
             {% block body %}{% endblock %}
         </body>
     </html>

把这段代码添加到布局里，这意味着所有继承布局的模板都要定义一个名为 ``conferences`` 的变量，这个变量由控制器创建并传递给模板。

由于我们只有 2 个控制器，你 *可能* 会要做下面的操作（不要照这样去改你的代码，我们很快会学习一种更好的方式）：

.. code-block:: diff
    :class: ignore

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -21,11 +21,12 @@ final class ConferenceController extends AbstractController
         }

         #[Route('/conference/{id}', name: 'conference')]
    -    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
    +    public function show(#[MapEntity] Conference $conference, CommentRepository $commentRepository, ConferenceRepository $conferenceRepository, #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0): Response
         {
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

             return $this->render('conference/show.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
                 'conference' => $conference,
                 'comments' => $paginator,
                 'previous' => $offset - CommentRepository::COMMENTS_PER_PAGE,

想象一下你必须更新几十个控制器，而且新的控制器里也要这样做。这不太现实。肯定有一个更好的方式。

Twig 支持全局变量。*全局变量* 在所有渲染的模板中可用。你可以在一个配置文件中定义它们，但这只适用于值不变的全局变量。若要添加一个 Twig 全局变量来引用所有会议，我们需要创建一个 *监听器*。

认识 Symfony 的事件
------------------------

.. index::
    single: Components;Event Dispatcher
    single: Event

Symfony 自带一个事件分发组件。分发器在某些特定的时刻 *分发* 某些 *事件*， *监听器* 则监听这些事件。 *监听器* 就是接入框架内部的钩子。

例如，一些事件允许你和 HTTP 请求的生命周期进行交互。在处理一个请求的过程中，分发器会分发一些事件，这可以发生在请求对象被创建的时候，或是在控制器即将执行的时候，也可以在应答对象就绪并准备发送的时候，还可以在一个异常抛出的时候。*监听器* 可以监听一个或多个事件，并且根据事件上下文执行一些逻辑。

事件是一些定义良好的扩展点，它使得这个框架适应于更多的场景，也更具扩展性。很多 Symfony 组件，比如 Security 组件、Messenger 组件、Workflow 组件和 Mailer 组件大量使用事件。

另一个自带的事件和监听器的实际案例就是命令的生命周期：你可以创建一个监听器，让它在 *任何* 命令执行之前运行一些代码。

任何包和 bundle 可以分发自己的事件，这使得它们的代码更具扩展性。

为了避免写一个配置文件来描述监听器要监听哪些事件，可以在监听器的类或方法上添加 ``#[AsEventListener]`` 属性。这个机制使得监听器可以自动注册到 Symfony 的事件分发器。

实现一个监听器
---------------------

.. index::
    single: Event;Listener
    single: Listener
    single: Command;make:listener

你对于使用 *Maker Bundle* 来生成代码应该牢记于心了，现在用它来生成一个监听器：

.. code-block:: terminal
    :class: answers(Symfony\\Component\\HttpKernel\\Event\\ControllerEvent)

    $ symfony console make:listener TwigEventListener

这个命令会问你要监听哪个事件。选择``Symfony\Component\HttpKernel\Event\ControllerEvent``事件，该事件在控制器被调用前那一刻被分发。这是注入``conferences``全局变量的最好时机，这样当控制器渲染模板时，Twig就可以使用它。按照下面的代码来更新你的监听器：

.. code-block:: diff
    :caption: patch_file

    --- i/src/EventListener/TwigEventListener.php
    +++ w/src/EventListener/TwigEventListener.php
    @@ -2,14 +2,22 @@

     namespace App\EventListener;

    +use App\Repository\ConferenceRepository;
     use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
     use Symfony\Component\HttpKernel\Event\ControllerEvent;
    +use Twig\Environment;

     final class TwigEventListener
     {
    +    public function __construct(
    +        private Environment $twig,
    +        private ConferenceRepository $conferenceRepository,
    +    ) {
    +    }
    +
         #[AsEventListener]
         public function onControllerEvent(ControllerEvent $event): void
         {
    -        // ...
    +        $this->twig->addGlobal('conferences', $this->conferenceRepository->findAll());
         }
     }

现在，你可以添加任意多的控制器：``conferences`` 变量在 Twig 里总是可用。

.. note::

    在以后的步骤中，我们会谈到一个性能好很多的替代方案。

按照年份和城市对会议进行排序
------------------------------------------

将会议按照年份排序可以让浏览变得更容易。我们可以创建一个定制方法来获取会议并对它们排序，但这里我们不用这个方式，而是去覆盖 ``findAll()`` 方法的默认实现，这样可以确保程序的所有地方都应用这个排序。

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/ConferenceRepository.php
    +++ w/src/Repository/ConferenceRepository.php
    @@ -16,6 +16,11 @@ class ConferenceRepository extends ServiceEntityRepository
             parent::__construct($registry, Conference::class);
         }

    +    public function findAll(): array
    +    {
    +        return $this->findBy([], ['year' => 'ASC', 'city' => 'ASC']);
    +    }
    +
         //    /**
         //     * @return Conference[] Returns an array of Conference objects
         //     */

在本步骤的结尾，网站看上去应该像这样：

.. figure:: screenshots/header.png
    :alt: /
    :align: center
    :figclass: with-browser

.. sidebar:: 深入学习

    * Symfony 应用中的 `请求-应答流`_；

    * `Symfony 自带的 HTTP 事件`_；

    * `Symfony 自带的 Console 事件`_。

.. _`请求-应答流`: https://symfony.com/doc/current/components/http_kernel.html#the-workflow-of-a-request
.. _`Symfony 自带的 HTTP 事件`: https://symfony.com/doc/current/reference/events.html
.. _`Symfony 自带的 Console 事件`: https://symfony.com/doc/current/components/console/events.html
