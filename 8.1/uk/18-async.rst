Перехід до асинхронності
==============================================

.. index::
    single: Async

Перевірка на спам, під час обробки відправленої форми, може призвести до певних проблем. Якщо модель ШІ працюватиме повільно, наш веб-сайт також уповільниться для користувачів. Але, що ще гірше, якщо ми натрапимо на тайм-аут, або модель буде недоступна, ми можемо втратити коментарі.

В ідеалі, ми маємо зберігати відправлені дані не публікуючи їх, відразу повертаючи відповідь. Перевірка на спам може бути виконана в іншому потоці.

Позначення коментарів
-----------------------------------------

.. index::
    single: Command;make:entity

Нам потрібно ввести стан (``state``) для коментарів: ``submitted`` (надіслано), ``spam`` (спам), та ``published`` (опубліковано).

Додайте властивість ``state`` у клас ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Ми також маємо переконатися, що за замовчуванням для властивості ``state`` встановлено значення ``submitted``:

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

Створіть міграцію бази даних:

.. code-block:: terminal

    $ symfony console make:migration

Модифікуйте міграцію, щоб оновити всі наявні коментарі, для яких буде встановлено стан ``published`` за замовчуванням:

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

Виконайте міграцію бази даних:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Оновіть логіку відображення, щоб уникнути появи неопублікованих коментарів на веб-сайті:

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

Оновіть конфігурацію EasyAdmin, щоб мати можливість бачити стан коментарів:

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

Не забудьте також оновити тестові фабрики: коментарі, створені ``CommentFactory``, мають бути за замовчуванням опублікованими, щоб з'являтися на сторінках конференцій (тест завжди може перевизначити стан, коли йому потрібен модерований коментар):

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

Для тестів контролера змоделюйте перевірку:

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

З тесту PHPUnit ви можете отримати будь-який сервіс із контейнера за допомогою ``self::$container->get()``; він також надає доступ до непублічних сервісів.

Опанування Messenger
------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Керування асинхронним кодом у Symfony — це завдання компонента Messenger:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Коли певна логіка має виконуватися асинхронно, відправте *повідомлення* у *шину повідомлень*. Шина зберігає повідомлення у *черзі* та негайно повертає відповідь, щоб потік операцій відновився якомога швидше.

*Споживач* працює безперервно, у фоновому режимі, читаючи нові повідомлення в черзі та виконуючи відповідну логіку. Споживач може працювати на тому ж сервері, що й веб-застосунок, або на окремому сервері.

Це дуже схоже на те, як обробляються HTTP-запити, за винятком того, що у нас немає відповідей.

Створення обробника повідомлень
------------------------------------------------------------

Повідомлення — це клас об’єкту даних, який не повинен містити жодної логіки. Він буде серіалізований для збереження у черзі, тому зберігайте лише "прості" дані, що можуть бути серіалізованими.

Створіть клас ``CommentMessage``:

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

У світі Messenger, у нас не контролери, а обробники повідомлень.

Створіть клас ``CommentMessageHandler`` у новому просторі імен ``App\MessageHandler``, який знатиме як обробляти повідомлення ``CommentMessage``:

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

``AsMessageHandler`` допомагає Symfony автоматично зареєструвати та налаштувати клас у якості обробника Messenger. За домовленістю, логіка обробника знаходиться у методі з назвою ``__invoke()``. Підказка типу ``CommentMessage``, в єдиному аргументі цього методу, повідомляє Messenger про те, який саме клас він буде обробляти.

Оновіть контролер, щоб використовувати нову систему:

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

Замість того щоб залежати від засобу перевірки на спам, тепер ми  відправляємо повідомлення до шини. Потім обробник вирішує, що з ним робити.

Ми досягли чогось несподіваного. Ми відокремили наш контролер від засобу перевірки на спам і перенесли логіку в новий клас обробник. Це ідеальний сценарій використання шини повідомлень. Перевірте код, він працює. Все як і раніше виконується синхронно, але код, напевно, вже став "краще".

Перехід до реальної асинхронності
---------------------------------------------------------------

За замовчуванням обробники викликаються синхронно. Щоб перейти до асинхронності, необхідно явно налаштувати, яку чергу використовувати для кожного обробника у файлі конфігурації ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

Конфігурація повідомляє шині про те, що екземпляри ``App\Message\CommentMessage`` слід відправляти в чергу ``async``, яка визначена за допомогою DSN (``MESSENGER_TRANSPORT_DSN``), що вказує на Doctrine, як налаштовано в `` .env``. Якщо говорити простою мовою, ми використовуємо PostgreSQL у якості черги для наших повідомлень.

.. tip::

    За лаштунками Symfony використовує вбудовану в PostgreSQL продуктивну, масштабовану і транзакційну систему pub/sub (``LISTEN``/``NOTIFY``). Ви також можете прочитати розділ про RabbitMQ, якщо хочете використовувати його замість PostgreSQL у якості брокера повідомлень.

Опрацювання повідомлень
---------------------------------------------

Якщо ви спробуєте відправити новий коментар, засіб перевірки на спам більше не викликатиметься. Додайте виклик ``error_log()`` у метод ``getSpamScore()``, щоб переконатися у цьому. Натомість повідомлення очікуватиме в черзі, в готовності до опрацювання певними процесами.

.. index::
    single: Command;messenger:consume

Як ви можете уявити, Symfony поставляється із командою споживача. Виконайте її:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Це має негайно опрацювати повідомлення, відправлене для надісланого коментаря:

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

Активність споживача повідомлень записується в журнал, але ви отримаєте миттєві повідомлення в консолі, передавши прапорець ``-vv``. Ви навіть зможете помітити виклик API OpenAI.

Щоб зупинити споживача, натисніть ``Ctrl+C``.

Запуск воркерів у фоновому режимі
--------------------------------------------------------------

Замість того щоб запускати споживача кожного разу, коли ми публікуємо коментар, і одразу ж зупиняти його, ми хочемо щоб він працював безперервно, не маючи занадто багато відкритих вікон або вкладок термінала.

Symfony CLI може керувати такими фоновими командами або воркерами, використовуючи прапорець демона (``-d``), команди ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Запустіть споживача  повідомлень ще раз, але відправте його у фоновий режим:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

Параметр ``--watch`` вказує Symfony, що команда має бути перезапущена щоразу, коли відбувається зміна файлової системи в каталогах ``config/``, ``src/``, ``templates/`` чи ``vendor/``.

.. note::

    Не використовуйте ``-vv`` тому, що ви дублюватимете повідомлення у ``server:log`` (повідомлення у журналі й консольні повідомлення).

Якщо споживач з якоїсь причини перестає працювати (обмеження пам'яті, помилка, ...), він буде перезапущений автоматично. Але якщо споживач перестає працювати занадто швидко, Symfony CLI не зможе його запустити.

.. index::
    single: Symfony CLI;server:log

Журнали можна переглянути за допомогою ``symfony server:log``, разом з усіма іншими журналами, що надходять із PHP, веб-сервера та застосунку:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Використовуйте команду ``server:status``, щоб вивести список всіх воркерів, що працюють у фоновому режимі, керованих для поточного проекту:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Щоб зупинити воркер, зупиніть веб-сервер або завершіть процес за PID, що наданий командою ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Повторна відправка повідомлень у разі помилки
-------------------------------------------------------------------------------------

Що робити, якщо база даних не працює під час опрацювання повідомлення? Це не впливає на користувачів, які відправляють коментарі, але повідомлення завершується невдало і перевірка на спам не проходить.

Messenger має механізм для повторної відправки у разі виникнення винятку під час обробки повідомлення:

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

Якщо під час обробки повідомлення виникає проблема, споживач повторить спробу 3 рази, перш ніж зупинитися. Але замість того щоб відкинути повідомлення, він перманентно збереже його у черзі ``failed``, яка використовує іншу таблицю бази даних.

Перевірте повідомлення з помилками й повторіть спробу за допомогою наступних команд:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Запуск воркерів у Upsun
--------------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Щоб опрацьовувати повідомлення з PostgreSQL, нам потрібно постійно виконувати команду ``messenger:consume``. У Upsun цю роль виконує *воркер*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Як і Symfony CLI, Upsun керує перезавантаженнями й журналами.

.. index::
    single: Symfony CLI;cloud:log

Щоб отримати журнали воркера, використовуйте:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Йдемо далі

    * `Навчальний посібник SymfonyCasts: Messenger`_;

    * Архітектура `корпоративної шини даних`_ і  `шаблон CQRS`_;

    * `Документація по Symfony Messenger`_;

.. _`Навчальний посібник SymfonyCasts: Messenger`: https://symfonycasts.com/screencast/messenger
.. _`корпоративної шини даних`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`шаблон CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`Документація по Symfony Messenger`: https://symfony.com/doc/current/messenger.html
