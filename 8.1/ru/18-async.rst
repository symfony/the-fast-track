Переход к асинхронности
============================================

.. index::
    single: Async

Во время проверки формы на наличие спама могут возникнуть проблемы. В случае, если ответ от модели ИИ будет долгим, наша страница с формой также станет работать медленнее. Худший сценарий — мы можем совсем потерять комментарии из-за достижения тайм-аута или недоступности модели.

В идеале нам нужно немедленно возвращать ответ на отправленную форму, а приложение должно сохранять данные без попытки их сразу же опубликовать. Проверить на спам можно и позже.

Установка статусов комментариев
------------------------------------------------------------

.. index::
    single: Command;make:entity

Реализуем статусы (``state``) для комментариев: ``submitted`` (отправлен), ``spam`` (спам) и ``published`` (опубликован).

Добавим свойство ``state`` в класс ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Убедимся, что для свойства ``state`` по умолчанию задан статус ``submitted``.

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

Создайте миграцию базы данных:

.. code-block:: terminal

    $ symfony console make:migration

Измените миграцию, чтобы все существующие комментарии были по умолчанию со статусом ``published``:

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

Выполните миграцию базы данных:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Обновите логику отображения, чтобы на странице показывались только опубликованные комментарии:

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

Обновите конфигурацию EasyAdmin, чтобы отображать статус комментария в административной панели:

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

Не забудьте также обновить тестовые фабрики: комментарии, создаваемые ``CommentFactory``, должны по умолчанию быть опубликованными, чтобы отображаться на страницах конференций (тест всегда может переопределить состояние, когда ему нужен комментарий на модерации):

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

Для тестирования контроллера смоделируйте проверку комментария:

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

В тесте PHPUnit вы можете получить любой сервис из контейнера с помощью метода ``self::getContainer()->get()``, который также даёт доступ к непубличным сервисам.

Внедрение компонента Messenger
-------------------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Компонент Messenger управляет асинхронным кодом в Symfony:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Для выполнения асинхронных задач, отправьте *сообщение* на *шину обмена сообщениями*. Шина сохраняет сообщение в *очередь* и немедленно возвращает ответ, чтобы исключить любые задержки в ваших приложениях.

*Получатель* работает постоянно в фоновом режиме, читая новые сообщения из очереди и выполняя соответствующие задачи. Получатель может работать как на том же сервере, где находится само приложение, так и на отдельном сервере.

Это напоминает обработку HTTP-запросов, за исключением того, что здесь нет ответов.

Создание обработчика сообщений
----------------------------------------------------------

Сообщение — это класс с данными, в котором не должно быть никакой логики. Он будет сериализован для хранения в очереди, поэтому держите в нем только "простые" сериализуемые данные.

Создайте класс ``CommentMessage``:

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

В мире Messenger нет контроллеров, есть только обработчики сообщений.

Создайте класс ``CommentMessageHandler`` в новом пространстве имён ``App\MessageHandler``, который  знает, как обрабатывать сообщения ``CommentMessage``:

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

``AsMessageHandler`` помогает Symfony понять, что класс нужно автоматически зарегистрировать и настроить в качестве обработчика Messenger. По соглашению логика обработчика находится в методе ``__invoke()``. Подсказка об используемом типе ``CommentMessage`` для единственного аргумента этого метода сообщает Messenger, какой класс он будет обрабатывать.

Обновите контроллер, чтобы использовать новую систему:

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

Теперь вместо того, чтобы зависеть от скорости работы антиспам-сервиса, мы отправляем сообщение на шину. Затем обработчик решает, что делать с этим сообщением.

Произошло что-то неожиданное: мы отделили наш контроллер от антиспам-сервиса и перенесли логику в новый класс — обработчик. Это идеальный вариант для использования шины. Проверьте код — всё работает. Операции всё ещё выполняются синхронно, но код, вероятно, уже стал "лучше".

Переходим к асинхронности по-настоящему
--------------------------------------------------------------------------

По умолчанию обработчики вызываются синхронно. Для  асинхронного вызова необходимо указать, какую очередь использовать для каждого обработчика в конфигурационном файле ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

Шина настроена на отсылку сообщений типа ``App\Message\CommentMessage`` в очередь ``async``, которая управляется Doctrine (согласно DSN-значению в переменной окружения ``MESSENGER_TRANSPORT_DSN`` из файла ``.env``). Проще говоря, мы используем PostgreSQL в качестве очереди для наших сообщений.

.. tip::

    За кулисами Symfony использует встроенную в PostgreSQL  производительную, масштабируемую и транзакционную систему подписки pub/sub (``LISTEN``/``NOTIFY``). Прочитайте главу о RabbitMQ, если хотите заменить брокер сообщений с  PostgreSQL на RabbitMQ.

Получение сообщений
-------------------------------------

При отправке нового комментария, проверка на спам больше не сработает. Чтобы убедиться в этом, добавьте вызов функции``error_log()`` в метод ``getSpamScore()``. Видно, что  сообщение находится в очереди и ждёт обработки.

.. index::
    single: Command;messenger:consume

Как вы понимаете, у Symfony есть команда для получения сообщений. Давайте выполним её сейчас:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Команда немедленно получит сообщение, отправленное для комментария:

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

Все действия получателя сообщений пишутся в логи, но эти же действия можно увидеть в консоли в реальном времени, если к команде добавить флаг ``-vv``. Вы даже сможете увидеть вызов к OpenAI API.

Для остановки получателя, нажмите ``Ctrl+C``.

Запуск воркеров в фоновом режиме
------------------------------------------------------------

Вместо того, чтобы запускать получателя каждый раз после отправки комментария и тут же его останавливать, хотелось бы, чтобы он работал непрерывно, и при этом не нужно было бы открывать слишком много окон или вкладок терминала.

Symfony CLI может управлять такими фоновыми командами или воркерами, если добавить флаг ( ``-d`` ) к команде ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Запустите получателя сообщений ещё раз, но теперь уже в фоновом режиме:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

При использовании параметра ``--watch`` Symfony перезапускает команду каждый раз при изменении файловой системы в директориях ``config/``, ``src/``, ``templates/`` или ``vendor/``.

.. note::

    Не используйте флаг ``-vv``, иначе получите дублирующие сообщения в логах команды ``server:log`` (логированные и консольные сообщения).

Если получатель перестаёт работать по какой-либо причине (недостаточно памяти, баг и т.д.), он будет перезапущен автоматически. Но Symfony CLI не перезапустит получателя в том случае, если получатель выдает ошибку сразу после запуска.

.. index::
    single: Symfony CLI;server:log

Логи можно просматривать через команду ``symfony server:log`` вместе с остальными логами, приходящими от PHP, веб-сервера и приложения:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Команда ``server:status`` выведет все фоновые воркеры для текущего проекта:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Чтобы остановить воркер, нужно либо остановить веб-сервер, либо принудительно завершить процесс по его PID после запуска команды ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Повторная обработка сообщений в случае ошибки
-------------------------------------------------------------------------------------

Что, если база данных будет недоступна во время получения сообщения? Это не отразится на пользователях, оправляющих комментарии, но сообщение завершится ошибкой и, как следствие, проверка на спам не будет выполнена.

У компонента Messenger есть механизм повторных попыток в случае возникновения исключений во время обработки сообщений:

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

Если при обработке сообщения возникнет ошибка, получатель повторит попытку ещё 3 раза, а затем остановится. Но вместо удаления получатель перенесёт сообщение в постоянное хранилище — очередь ``failed``, которая использует другую таблицу базу данных.

Просматривать сообщения, давшие сбой и повторить попытку их обработки можно с помощью следующих команд:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Выполнение воркеров на Upsun
------------------------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Чтобы получать сообщения от PostgreSQL, нам нужно постоянно запускать команду ``messenger:consume``. В Upsun для этой задачи отведена специальная роль — *воркер*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Подобно Symfony CLI, Upsun позволяет управлять перезагрузками и логами.

.. index::
    single: Symfony CLI;cloud:log

Чтобы посмотреть логи воркера, воспользуйтесь командой ниже:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Двигаемся дальше

    * `Видеокурс по Messenger на SymfonyCasts`_;

    * `Архитектура сервисной шины предприятия`_ и `шаблон CQRS`_;

    * `Документация по Symfony Messenger`_;

.. _`Видеокурс по Messenger на SymfonyCasts`: https://symfonycasts.com/screencast/messenger
.. _`Архитектура сервисной шины предприятия`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`шаблон CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`Документация по Symfony Messenger`: https://symfony.com/doc/current/messenger.html
