使用异步
============

.. index::
    single: Async

在处理表单提交时来检查垃圾信息会造成一些问题。如果 AI 模型变得很慢，我们的网站对用户的响应也会很慢。但更糟的是，如果调用超时了，或者模型不可用，那我们可能会丢失评论。

理想情况是，你应该存储提交的数据，但先不发布它，并且立刻返回一个应答。检查垃圾信息可以之后进行（带外执行）。

对评论进行标记
---------------------

.. index::
    single: Command;make:entity

我们要为评论引入 ``state`` 信息，它可以取以下值之一：``submitted``、``spam`` 或 ``published``。

把 ``state`` 属性加入 ``Comment`` 类：

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

我们也要确保 ``state`` 的默认值设为 ``submitted``：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Entity/Comment.php
    +++ w/src/Entity/Comment.php
    @@ -39,8 +39,8 @@ class Comment
         #[ORM\Column(length: 255, nullable: true)]
         private ?string $photoFilename = null;

    -    #[ORM\Column(length: 255)]
    -    private ?string $state = null;
    +    #[ORM\Column(length: 255, options: ['default' => 'submitted'])]
    +    private ?string $state = 'submitted';

         public function getId(): ?int
         {

.. index::
    single: Command;make:migration

创建一个数据库结构迁移：

.. code-block:: terminal

    $ symfony console make:migration

修改这个迁移，让已有评论的状态默认设为 ``published``：

.. code-block:: diff
    :caption: patch_file

    --- i/migrations/Version00000000000000.php
    +++ w/migrations/Version00000000000000.php
    @@ -21,6 +21,7 @@ final class Version00000000000000 extends AbstractMigration
         {
             // this up() migration is auto-generated, please modify it to your needs
             $this->addSql('ALTER TABLE comment ADD state VARCHAR(255) DEFAULT \'submitted\' NOT NULL');
    +        $this->addSql("UPDATE comment SET state='published'");
         }

         public function down(Schema $schema): void

.. index::
    single: Command;doctrine:migrations:migrate

更新数据库结构：

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

更新评论的展示逻辑，让未发布的评论不出现在前端：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -24,7 +24,9 @@ class CommentRepository extends ServiceEntityRepository
         {
             $query = $this->createQueryBuilder('c')
                 ->andWhere('c.conference = :conference')
    +            ->andWhere('c.state = :state')
                 ->setParameter('conference', $conference)
    +            ->setParameter('state', 'published')
                 ->orderBy('c.createdAt', 'DESC')
                 ->setMaxResults(self::COMMENTS_PER_PAGE)
                 ->setFirstResult($offset)

更新 EasyAdmin 的配置，这样我们可以看到评论的状态：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/Admin/CommentCrudController.php
    +++ w/src/Controller/Admin/CommentCrudController.php
    @@ -53,6 +53,7 @@ class CommentCrudController extends AbstractCrudController
                 ->setLabel('Photo')
                 ->onlyOnIndex()
             ;
    +        yield TextField::new('state');

             $createdAt = DateTimeField::new('createdAt')->setFormTypeOptions([
                 'years' => range(date('Y'), date('Y') + 5),

别忘了也要更新测试工厂：``CommentFactory`` 创建的评论默认应该是已发布的，这样它们才会出现在会议页面上（当某个测试需要一条被审核的评论时，它总是可以覆盖这个状态）：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Factory/CommentFactory.php
    +++ w/src/Factory/CommentFactory.php
    @@ -38,6 +38,7 @@ final class CommentFactory extends PersistentObjectFactory
                 'conference' => ConferenceFactory::new(),
                 'createdAt' => \DateTimeImmutable::createFromMutable(self::faker()->dateTime()),
                 'email' => self::faker()->email(),
    +            'state' => 'published',
                 'text' => self::faker()->text(),
             ];
         }

.. index::
    single: Test;Container
    single: Container;Test

在控制器的测试里，模拟验证：

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -4,6 +4,8 @@ namespace App\Tests\Controller;

     use App\Factory\CommentFactory;
     use App\Factory\ConferenceFactory;
    +use App\Repository\CommentRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
     use Zenstruck\Foundry\Test\Factories;
     use Zenstruck\Foundry\Test\ResetDatabase;
    @@ -33,10 +35,16 @@ class ConferenceControllerTest extends WebTestCase
             $client->submitForm('Submit', [
                 'comment[author]' => 'Fabien',
                 'comment[text]' => 'Some feedback from an automated functional test',
    -            'comment[email]' => 'me@automat.ed',
    +            'comment[email]' => $email = 'me@automat.ed',
                 'comment[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
             ]);
             $this->assertResponseRedirects();
    +
    +        // simulate comment validation
    +        $comment = self::getContainer()->get(CommentRepository::class)->findOneByEmail($email);
    +        $comment->setState('published');
    +        self::getContainer()->get(EntityManagerInterface::class)->flush();
    +
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }

在 PHPUnit 测试中，你可以用 ``self::getContainer()->get()`` 获取容器里的任何服务；你也可以用它访问非公共服务。

理解 Messenger
----------------

.. index::
    single: Messenger
    single: Components;Messenger

在 Symfony 中由 Messenger 组件来管理异步代码：

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

当需要异步执行一些逻辑时，发送一个 *消息* 给 *消息总线*。总线会在一个 *队列* 中存储消息，并且立即返回，这样执行流程就能尽快恢复。

一个 *消费者* 程序会在后台持续运行，它会读取队列里的消息，并且执行相关逻辑。消费者程序可以和 web 应用程序运行在同一个服务器上，也可以运行在另一个服务器上。

它和 HTTP 请求的处理方式很像，只是它不会返回应答。

编写一个消息处理器
---------------------------

一个消息是一个仅包含数据的类，它不含任何逻辑。它会被序列化，然后保存在队列中，所以它只包含“简单”的可序列化数据。

创建 ``CommentMessage`` 类：

.. code-block:: php
    :caption: src/Message/CommentMessage.php

    namespace App\Message;

    class CommentMessage
    {
        public function __construct(
            private int $id,
            private array $context = [],
        ) {
        }

        public function getId(): int
        {
            return $this->id;
        }

        public function getContext(): array
        {
            return $this->context;
        }
    }

在 Messenger 的世界里，我们没有控制器，但有消息处理器。

在新的 ``App\MessageHandler`` 命名空间下创建 ``CommentMessageHandler`` 类，它知道如何处理 ``CommentMessage`` 类的消息：

.. code-block:: php
    :caption: src/MessageHandler/CommentMessageHandler.php

    namespace App\MessageHandler;

    use App\Message\CommentMessage;
    use App\Repository\CommentRepository;
    use App\SpamChecker;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Component\Messenger\Attribute\AsMessageHandler;

    #[AsMessageHandler]
    class CommentMessageHandler
    {
        public function __construct(
            private EntityManagerInterface $entityManager,
            private SpamChecker $spamChecker,
            private CommentRepository $commentRepository,
        ) {
        }

        public function __invoke(CommentMessage $message): void
        {
            $comment = $this->commentRepository->find($message->getId());
            if (!$comment) {
                return;
            }

            if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
                $comment->setState('spam');
            } else {
                $comment->setState('published');
            }

            $this->entityManager->flush();
        }
    }

``AsMessageHandler`` 帮助 Symfony 自动注册和自动配置该类为一个消息处理器。根据惯例，处理逻辑放在名为 ``__invoke()`` 的方法中。该方法中唯一参数的 ``CommentMessage`` 类型提示告诉 Messenger 它要处理的类是哪一个。

更新控制器来使用新的系统：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -5,22 +5,24 @@ namespace App\Controller;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use App\Form\CommentType;
    +use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
    +use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         public function __construct(
             private EntityManagerInterface $entityManager,
    +        private MessageBusInterface $bus,
         ) {
         }

    @@ -35,8 +37,7 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    -        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -50,6 +51,7 @@ final class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +            $this->entityManager->flush();

                 $context = [
                     'user_ip' => $request->getClientIp(),
    @@ -57,11 +59,7 @@ final class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    -                throw new \RuntimeException('Blatant spam, go away!');
    -            }
    -
    -            $this->entityManager->flush();
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

我们现在分发一个消息到总线上，而不是依赖于垃圾信息检查器。然后消息处理器会决定怎么处理消息。

这达到了一些意想不到的效果。我们把控制器和垃圾信息检查器进行了解耦，把逻辑移到了消息处理器这个新的类中。这是一个使用总线的完美案例。测试一下代码，它能通过。所有的执行仍然是同步的，但代码大概已经变得“更好”了。

真正走向异步
------------------

默认情况下，消息处理器都是同步调用。为了使用异步方式，你需要在 ``config/packages/messenger.yaml`` 配置文件中为每个消息处理器显式地配置要用到的队列：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

该配置告诉总线把 ``App\Message\CommentMessage`` 类的实例发送到 ``async`` 队列，这个队列由 DSN （``MESSENGER_TRANSPORT_DSN``）变量定义，它指向了 Doctrine，这是在 ``.env`` 文件中配置的。用大白话说，我们使用 PostgreSQL 作为消息队列。

.. tip::

    在幕后，Symfony 使用了 PostgreSQL 的发布/订阅系统（``LISTEN``/``NOTIFY``），这个系统是 PostgreSQL 内置的、性能优越、具备可升缩性并且支持事务。如果你想要用 RabbitMQ 作为消息代理来取代 PostgreSQL，你可以阅读关于 RabbitMQ 的那一章。

用消费者程序处理消息
------------------------------

如果你试着提交一个新评论，垃圾检查器不再被调用。在 ``getSpamScore()`` 方法中加上一行 ``error_log()`` 调用来进行验证。运行结果是，消息会在消息队列中等待，直到有消费者进程来处理。

.. index::
    single: Command;messenger:consume

你可以想象得到，Symfony 自带了一个消费者命令。现在来运行它：

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

它会立刻处理掉那个表单提交后分发的消息：

.. code-block:: text
    :class: ignore

     [OK] Consuming messages from transports "async".

     // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

     // Quit the worker with CONTROL-C.

    11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
    11:30:20 INFO      [http_client] Request: "POST https://api.openai.com/v1/responses"
    11:30:20 INFO      [http_client] Response: "200 https://api.openai.com/v1/responses"
    11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
    11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]

日志会记录消息的处理，但你可以用 ``-vv`` 选项在终端获得即时反馈。你甚至应该可以在日志中发现调用 OpenAI API 的记录。

按 ``Ctrl+C`` 可以停止消费者程序。

在后台运行 worker 进程
-----------------------------

我们想要在不打开很多终端的情况下就可以让消费者程序持续运行，而不是每次提交一个评论时打开它，提交完后再立刻关闭它。

``symfony`` 命令在执行 ``run`` 命令时，可以通过守护进程选项（``-d``）来管理这种后台运行的命令或工作进程。

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

再执行一次消费者程序，但在后台运行它：

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

``--watch`` 选项告诉 Symfony，每当 ``config/``、``src/``、``templates/`` 或 ``vendor/`` 这些目录下有文件系统的改动时，这个命令必须重启。

.. note::

    不要使用 ``-vv`` 选项，否则你会在执行 ``server:log`` 时信息会重复（日志信息和控制台信息）。

如果消费者程序因为一些原因停止运行了（内存限制，程序错误等等），它会被自动重启。如果运行消费者程序失败得太频繁，``symfony`` 命令就会放弃运行。

.. index::
    single: Symfony CLI;server:log

日志通过 ``symfony server:log`` 流式输出，它会整合所有来自 PHP、web 服务器和应用程序的其它日志：

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

用 ``server:status`` 命令来列出当前项目所管理的全部后台 worker 进程：

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

要停止一个 worker 进程，需要停止 web 服务器，或者根据 ``server:status`` 命令显示的进程 ID 杀死对应进程：

.. code-block:: terminal
    :class: ignore

    $ kill 15774

对失败的消息进行重试
------------------------------

如果处理消息时数据库下线了怎么办？提交评论的人不会受影响，但这个消息会失败，垃圾信息检查也没有做。

Messenger 针对处理消息时抛出异常有一个重试机制：

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 3,12-15
    :class: ignore

    framework:
        messenger:
            failure_transport: failed

            transports:
                # https://symfony.com/doc/current/messenger.html#transport-configuration
                async:
                    dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                    options:
                        use_notify: true
                        check_delayed_interval: 1000
                    retry_strategy:
                        max_retries: 3
                        multiplier: 2
                failed: 'doctrine://default?queue_name=failed'
                # sync: 'sync://'

.. index::
    single: Command;messenger:failed:show
    single: Command;messenger:failed:retry

当处理消息时出现了问题，消费者程序会在放弃前重试 3 次。放弃后不会丢掉消息，而是把消息永久存储在 ``failed`` 队列中，这个队列使用了另一个数据库表。

查看处理失败的消息，用以下命令来重试处理：

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

在 Upsun 上运行 worker 进程
----------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

我们需要持续运行 ``messenger:consume`` 命令来处理 PostgreSQL 队列中的消息。在 Upsun 上，这是 *worker 进程* 的角色：

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

就像本地 ``symfony`` 命令一样，Upsun 会管理重启和日志。

.. index::
    single: Symfony CLI;cloud:logs

用这个命令来查看 worker 进程中的日志：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --worker=messages all

.. sidebar:: 深入学习

    * `SymfonyCasts 的 Messenger 教程`_；

    * `企业服务总线`_ 架构和 `CQRS 模式`_；

    * `Symfony 的 Messenger 文档`_；

.. _`SymfonyCasts 的 Messenger 教程`: https://symfonycasts.com/screencast/messenger
.. _`企业服务总线`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS 模式`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony 的 Messenger 文档`: https://symfony.com/doc/current/messenger.html
