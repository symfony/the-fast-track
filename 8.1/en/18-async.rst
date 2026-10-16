Going Async
===========

.. index::
    single: Async

Checking for spam during the handling of the form submission might lead to some problems. If the AI model becomes slow, our website will also be slow for users. But even worse, if we hit a timeout or if the model is unavailable, we might lose comments.

Ideally, we should store the submitted data without publishing it, and immediately return a response. Checking for spam can then be done out of band.

Flagging Comments
-----------------

.. index::
    single: Command;make:entity

We need to introduce a ``state`` for comments: ``submitted``, ``spam``, and ``published``.

Add the ``state`` property to the ``Comment`` class:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

We should also make sure that, by default, the ``state`` is set to ``submitted``:

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

Create a database migration:

.. code-block:: terminal

    $ symfony console make:migration

Modify the migration to update all existing comments to be ``published`` by default:

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

Migrate the database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Update the display logic to avoid non-published comments from appearing on the frontend:

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

Update the EasyAdmin configuration to be able to see the comment's state:

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

Don't forget to also update the test factories: comments created by ``CommentFactory`` should be published by default so that they show up on the conference pages (a test can always override the state when it needs a moderated comment):

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

For the controller tests, simulate the validation:

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

From a PHPUnit test, you can get any service from the container via ``self::getContainer()->get()``; it also gives access to non-public services.

Understanding Messenger
-----------------------

.. index::
    single: Messenger
    single: Components;Messenger

Managing asynchronous code with Symfony is the job of the Messenger Component:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

When some logic should be executed asynchronously, send a *message* to a *messenger bus*. The bus stores the message in a *queue* and returns immediately to let the flow of operations resume as fast as possible.

A *consumer* runs continuously in the background to read new messages on the queue and execute the associated logic. The consumer can run on the same server as the web application or on a separate one.

It is very similar to the way HTTP requests are handled, except that we don't have responses.

Coding a Message Handler
------------------------

A message is a data object class that should not hold any logic. It will be serialized to be stored in a queue, so only store "simple" serializable data.

Create the ``CommentMessage`` class:

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

In the Messenger world, we don't have controllers, but message handlers.

Create a ``CommentMessageHandler`` class under a new ``App\MessageHandler`` namespace that knows how to handle ``CommentMessage`` messages:

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

``AsMessageHandler`` helps Symfony auto-register and auto-configure the class as a Messenger handler. By convention, the logic of a handler lives in a method called ``__invoke()``. The ``CommentMessage`` type hint on this method's sole argument tells Messenger which class this will handle.

Update the controller to use the new system:

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

Instead of depending on the Spam Checker, we now dispatch a message on the bus. The handler then decides what to do with it.

We have achieved something unexpected. We have decoupled our controller from the Spam Checker and moved the logic to a new class, the handler. It is a perfect use case for the bus. Test the code, it works. Everything is still done synchronously, but the code is probably already "better".

Going Async for Real
--------------------

By default, handlers are called synchronously. To go async, you need to explicitly configure which queue to use for each handler in the ``config/packages/messenger.yaml`` configuration file:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

The configuration tells the bus to send instances of ``App\Message\CommentMessage`` to the ``async`` queue, which is defined by a DSN (``MESSENGER_TRANSPORT_DSN``), which points to Doctrine as configured in ``.env``. In plain English, we are using PostgreSQL as a queue for our messages.

.. tip::

    Behind the scenes, Symfony uses the PostgreSQL builtin, performant, scalable, and transactional pub/sub system (``LISTEN``/``NOTIFY``). You can also read the RabbitMQ chapter if you want to use it instead of PostgreSQL as a message broker.

Consuming Messages
------------------

If you try to submit a new comment, the spam checker won't be called anymore. Add an ``error_log()`` call in the ``getSpamScore()`` method to confirm. Instead, a message is waiting in the queue, ready to be consumed by some processes.

.. index::
    single: Command;messenger:consume

As you might imagine, Symfony comes with a consumer command. Run it now:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

It should immediately consume the message dispatched for the submitted comment:

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

The message consumer activity is logged, but you get instant feedback on the console by passing the ``-vv`` flag. You should even be able to spot the call to the OpenAI API.

To stop the consumer, press ``Ctrl+C``.

Running Workers in the Background
---------------------------------

Instead of launching the consumer every time we post a comment and stopping it immediately after, we want to run it continuously without having too many terminal windows or tabs open.

The Symfony CLI can manage such background commands or workers by using the daemon flag (``-d``) on the ``run`` command.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Run the message consumer again, but send it in the background:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

The ``--watch`` option tells Symfony that the command must be restarted whenever there is a filesystem change in the ``config/``, ``src/``, ``templates/``, or ``vendor/`` directories.

.. note::

    Do not use ``-vv`` as you would have duplicated messages in ``server:log`` (logged messages and console messages).

If the consumer stops working for some reason (memory limit, bug, ...), it will be restarted automatically. And if the consumer fails too fast, the Symfony CLI will give up.

.. index::
    single: Symfony CLI;server:log

Logs are streamed via ``symfony server:log`` with all the other logs coming from PHP, the web server, and the application:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Use the ``server:status`` command to list all background workers managed for the current project:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

To stop a worker, stop the web server or kill the PID given by the ``server:status`` command:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Retrying Failed Messages
------------------------

What if the database is down while consuming a message? There is no impact for people submitting comments, but the message fails and spam is not checked.

Messenger has a retry mechanism for when an exception occurs while handling a message:

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

If a problem occurs while handling a message, the consumer will retry 3 times before giving up. But instead of discarding the message, it will store it permanently in the ``failed`` queue, which uses another database table.

Inspect failed messages and retry them via the following commands:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Running Workers on Upsun
------------------------

.. index::
    single: Upsun;Workers
    single: Workers

To consume messages from PostgreSQL, we need to run the ``messenger:consume`` command continuously. On Upsun, this is the role of a *worker*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Like for the Symfony CLI, Upsun manages restarts and logs.

.. index::
    single: Symfony CLI;cloud:log

To get logs for a worker, use:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Going Further

    * `SymfonyCasts Messenger tutorial`_;

    * The `Enterprise service bus`_ architecture and the `CQRS pattern`_;

    * The `Symfony Messenger docs`_;

.. _`SymfonyCasts Messenger tutorial`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS pattern`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony Messenger docs`: https://symfony.com/doc/current/messenger.html
