پیش به سوی ناهمزمانی (Async)
=============================================

.. index::
    single: Async

بررسی هرز بودن محتوا در هنگام ارسال فرم ممکن است موجب مشکلاتی شود. اگر مدل هوش مصنوعی کند شود، وب‌سایت ما نیز برای کاربران کند می‌شود. اما بدتر از آن، اگر مهلت (timeout) پایان یابد یا مدل در دسترس نباشد، ممکن است ما کامنت‌ها را از دست بدهیم.

به صورت ایده‌آل ما باید داده‌های ارسالی را بدون انتشار آن ذخیره کنیم و به سرعت پاسخ را برگردانیم. سپس بررسی هرز بودن محتوا می‌تواند به صورت مجزا (out of band) صورت پذیرد.

پرچم‌گذاری کامنت‌ها
---------------------------------------

.. index::
    single: Command;make:entity

ما نیاز داریم که یک ``state`` برای کامنت‌ها معرفی کنیم: ``submitted``، ``spam`` و ``published``.

ویژگی ``state`` را به کلاس ``Comment`` اضافه کنید:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

همچنین باید اطمینان یابیم که به صورت پیش‌فرض،‌ ``state`` با مقدار ``submitted`` تنظیم شده است:

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

migration پایگاه‌داده را ایجاد کنید:

.. code-block:: terminal

    $ symfony console make:migration

migration را اصلاح کنید تا تمام کامنت‌های موجود را با مقدار پیش‌فرض ``published`` به‌روزرسانی نماید:

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

پایگاه‌داده را Migrate کنید:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

منطق نمایش را به‌روزرسانی کنید تا از ظاهرشدن کامنت‌های منتشرنشده در جلوی صحنه جلوگیری شود:

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

پیکربندی EasyAdmin را به‌روزرسانی کنید تا قادر به مشاهده‌ی وضعیت کامنت‌ها باشیم:

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

به‌روزرسانی factoryهای آزمون را فراموش نکنید: کامنت‌هایی که توسط ``CommentFactory`` ساخته می‌شوند باید به‌صورت پیش‌فرض منتشر (published) باشند تا روی صفحات کنفرانس نمایش داده شوند (یک آزمون همیشه می‌تواند هنگام نیاز به یک کامنت تعدیل‌شده، وضعیت را بازنویسی کند):

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

برای آزمون‌های کنترلر، اعتبارسنجی را شبیه‌سازی کنید:

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

از درون یک آزمون PHPUnit، می‌توانید هر سرویسی را از کانتینر و از طریق ``self::getContainer()->get()`` دریافت کنید؛ همچنین این روش دسترسی به سرویس‌های غیر عمومی را نیز ممکن می‌کند.

درک کردن پیغام‌رسان (Messenger)
-------------------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

مدیریت کدهای غیرهمزمان در سیمفونی بر عهده‌ی کامپوننت پیغام‌رسان (Messenger) است:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

زمانی که لازم باشد یک منطق به صورت غیرهمزمان اجرا گردد، یک *پیغام (message)* برای *گذرگاه پیغام‌رسان (messenger bus)* ارسال کنید. گذرگاه، پیغام را در یک *صف(queue)* ذخیره کرده و به صورت آنی باز می‌گرداند تا جریان عملیات بتوان به سریع‌ترین وجه ممکن ادامه یابد.

یک *مصرف‌کننده (consumer)* به صورت مستمر در پس‌زمینه پیغام‌های جدید درون صف را می‌خواند و منطق مربوطه را اجرا می‌کند. مصرف‌کننده می‌تواند بر روی همان سرور اپلیکیشن وب یا بر روی یک سرور مجزا اجرا گردد.

این بسیار شبیه به شیوه‌ی رسیدگی به درخواست‌های HTTP است، با این تفاوت که دیگر واکنشی نداریم.

کدنویسی یک رسیدگی‌کننده به پیغام (Message Handler)
-------------------------------------------------------------------------------

پیغام یک شیء داده‌ای است که نباید حاوی هیچ منطقی باشد. پیغام برای ذخیره‌شدن در صف، serialize می‌شود، بنابراین تنها داده‌های «ساده‌ی» قابل serialize‌کردن را ذخیره کنید.

یک کلاس ``CommentMessage`` ایجاد کنید:

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

در دنیای پیغام‌رسان، ما کنترلر نداریم، بلکه رسیدگی‌کنندگان به پیغام (message handlers) را داریم.

یک کلاس ``CommentMessageHandler`` که می‌داند چگونه به پیغام‌های ``CommentMessage`` رسیدگی کند را در فضای‌نام (namespace) جدید ``App\MessageHandler`` ایجاد کنید:

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

``AsMessageHandler`` به سیمفونی کمک می‌کند تا به صورت خودکار کلاس را به عنوان یک رسیدگی‌کننده‌ی پیغام‌رسان ثبت و پیکربندی کند. بر اساس قرارداد، منطق یک رسیدگی‌کننده در درون متدی که ``__invoke()`` نامیده می‌شود،‌ قرار می‌گیرد. راهنمای نوعِ (type hint) ``CommentMessage`` بر روی تنها آرگمان این متد، به پیغام‌رسان می‌گوید که این کلاس، به چه کلاسی (چه نوع پیغامی) رسیدگی می‌کند.

کنترلر را به‌روزرسانی کنید تا از سیستم جدید استفاده کند:

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

ما حالا به جای تکیه به بررسی‌کنند‌ه‌ی داده‌های هرز، یک پیغام را به گذرگاه اعزام می‌کنیم. سپس رسیدگی‌کننده تصمیم می‌گیرد که با آن چه کار کند.

ما به چیزی غیرمنتظره دست یافتیم. ما کنترلرمان را از بررسی‌کننده‌ی داده‌ی هرز (Spam Checker) مجزا کردیم و منطق آن را به یک کلاس جدید یعنی رسیدگی‌کننده (handler) انتقال دادیم. این یک نمونه‌ی عالی از کارکرد گذرگاه است. کد را بیازمایید، کار می‌کند. هنوز همه چیز به صورت همزمان کار می‌کند، اما احتمالاً الان کد «بهتر» است.

پیش به سوی ناهمزمانی واقعی
------------------------------------------------

به صورت پیش‌فرض، رسیدگی‌کننده‌ها به صورت همزمان اجرا می‌شوند. برای ناهمزمان کردن، لازم دارید که در فایل پیکربندی ``config/packages/messenger.yaml``، به صورت صریح مشخص کنید که برای هر رسیدگی‌کننده باید از کدام صف استفاده شود:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

پیکربندی به گذرگاه می‌گوید که نمونه‌های ``App\Message\CommentMessage`` را به صف ``async`` ارسال کند که توسط یک DSN (``MESSENGER_TRANSPORT_DSN``) تعریف شده و همان‌طور که در ``.env`` پیکربندی شده، به Doctrine اشاره می‌کند. به زبان ساده، ما از PostgreSQL به عنوان یک صف برای پیغام‌هایمان استفاده می‌کنیم.

.. tip::

    در پشت صحنه، سیمفونی از سامانه‌ی توکار، کارا، مقیاس‌پذیر و تراکنشی pub/sub در PostgreSQL (یعنی ``LISTEN``/``NOTIFY``) استفاده می‌کند. اگر می‌خواهید به جای PostgreSQL از RabbitMQ به عنوان دلال پیغام استفاده کنید، می‌توانید فصل مربوط به RabbitMQ را نیز بخوانید.

مصرف‌کردن پیغام‌ها
-------------------------------------

اگر سعی کنید که یک کامنت جدید ارسال کنید، دیگر بررسی‌کننده‌ی محتوای هرز اجرا نمی‌شود. برای تأیید این امر، یک فراخوانی ``error_log()`` به متد ``getSpamScore()`` بیافزایید. به جای آن، یک پیغام در صف در حال انتظار است و آمادگی دارد تا توسط یک پردازشگر مصرف شود.

.. index::
    single: Command;messenger:consume

همانطور که احتمالاً تصور کرده‌اید، سیمفونی یک فرمان مصرف‌کننده به همراه دارد. حالا آن را اجرا کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

این فرمان باید بی‌درنگ پیغامی را که برای کامنت ارسال‌شده اعزام گردیده است، مصرف کند:

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

فعالیت مصرف‌کننده‌ی پیغام، log شده است. اما شما با دادن پرچم ``-vv``، به صورت آنی در کنسول بازخورد می‌گیرید. حتی شما باید قادر به تشخیص فراخوانی API مربوط به OpenAI نیز باشید.

برای متوقف کردن مصرف‌کننده، ``Ctrl+C`` را فشار دهید.

اجرای کارگرها (Workers) در پس‌زمینه
----------------------------------------------------------

به جای اینکه هر بار پس از ارسال کامنت، مصرف‌کننده را اجرا و به محض پایان اجرا، آن را متوقف کنیم، ما می‌خواهیم که آن را به صورت پیوسته و بدون آنکه لازم  به بازکردن تعداد زیادی پنجره‌ یا تب ترمینال باشد، اجرا کنیم.

رابط خط فرمان سیمفونی می‌تواند با پرچم شبه (``-d``) بر روی فرمان ``run``، چنین فرمان‌ها یا کارگر‌هایِ پس‌زمینه‌ای را اجرا کند.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

مجدداً مصرف‌کننده‌ی پیغام را اجرا کنید، اما آن را به پس‌زمینه بفرستید:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

گزینه‌ی ``--watch`` به سیمفونی می‌گوید که هرگاه تغییری از نوع فایل‌سیستم در پوشه‌های ``config/``، ``src/``، ``templates/`` یا ``vendor/`` بوجود آید، باید این فرمان را مجدداً راه‌اندازی نماید.

.. note::

    از پرچم ``-vv`` استفاده نکنید چرا که موجب پیغام‌های تکراری در ``server:log`` می‌شود (پیغام‌های لاگ‌شده و پیغام‌های کنسول).

اگر مصرف‌کننده به دلیلی (محدودیت حافظه، باگ و ...) از کار بیافتد، به صورت خودکار راه‌اندازی مجدد خواهد شد. و اگر مصرف‌کننده مجدداً خیلی سریع شکست بخورد، رابط خط فرمان سیمفونی، بازراه‌اندازی آن را رها خواهد کرد.

.. index::
    single: Symfony CLI;server:log

لاگ‌ها از طریق ``symfony server:log`` به همراه سایر لاگ‌هایی که از PHP، وب سرور و اپلیکیشن می‌آیند، جریان می‌یابند:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

از فرمان ``server:status`` استفاده کنید تا تمام کارگرهای پس‌زمینه‌ی به‌کاررفته برای پروژه‌ی فعلی لیست شوند:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

برای متوقف‌کردن یک کارگر، وب سرور را متوقف کنید یا PID داده‌شده توسط فرمان ``server:status`` را بکشید:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

تلاش مجدد برای پیغام‌های شکست‌خورده
--------------------------------------------------------------------

چه پیش می‌آید اگر هنگام مصرف پیغام، پایگاه‌داده خراب باشد؟ برای افراد ارسال‌کننده‌ی کامنت مشکلی ایجاد نمی‌شود، اما پیغام شکست می‌خورد و هرزبودن داده بررسی نمی‌گردد.

پیغام‌رسان دارای یک مکانیسم تلاش مجدد برای زمان‌هایی است که یک استثناء هنگام رسیدگی به پیغام رخ می‌دهد:

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

اگر هنگام رسیدگی به پیغام، مشکلی رخ دهد، مصرف‌کننده ۳ بار تلاش مجدد می‌کند و پس از آن از تلاش بیشتر دست می‌کشد. اما به جای دورانداختن پیغام، آن را به صورت دائمی در صف ``failed`` که از یک جدول دیگر پایگاه‌داده استفاده می‌کند، ذخیره می‌کند.

با کمک فرمان‌های زیر، پیغام‌های شکست‌خورده را بررسی کرده و مجدداً برای توفیق آن‌ها تلاش کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

اجرای کارگرها در Upsun
-------------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

برای مصرف پیغام‌ها از PostgreSQL، ما نیاز داریم تا فرمان ``messenger:consume`` را به صورت مستمر اجرا کنیم. در Upsun، این نقشِ یک *کارگر (worker)* است:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

همچون رابط خط فرمان سیمفونی، اینجا نیز Upsun بازراه‌اندازی‌ها و لاگ‌ها را مدیریت می‌کند.

.. index::
    single: Symfony CLI;cloud:logs

برای گرفتن لا‌گ‌های یک کارگر، از این استفاده کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --worker=messages all

.. sidebar:: بیشتر بدانید

    * `آموزش تصویری پیغام‌رسان در SymfonyCasts`_؛

    * معماری `گذرگاه سرویس سازمانی (ESB)`_ و `الگوی CQRS`_؛

    * `مستندات پیغام‌رسان سیمفونی`_؛

.. _`آموزش تصویری پیغام‌رسان در SymfonyCasts`: https://symfonycasts.com/screencast/messenger
.. _`گذرگاه سرویس سازمانی (ESB)`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`الگوی CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`مستندات پیغام‌رسان سیمفونی`: https://symfony.com/doc/current/messenger.html
