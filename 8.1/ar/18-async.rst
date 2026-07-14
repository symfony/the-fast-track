الذهاب المتزامن
=============================

.. index::
    single: Async

التحقق من البيانات التطفلية (Spam) أثناء معالجة الاستمارة (الفورم Form) قد يؤدي لبعض المشاكل. لو ان نموذج الذكاء الاصطناعي (AI model) اصبح بطيئاً، كذلك موقعنا سوف يصبح بطئ بالنسبة للمستخدمين. لكن أسوء من ذلك، اذا وصلنا إلي وقت مستقطع (timeout) أو أصبح النموذج غير متاح، من الممكن ان نخسر التعليقات (comments).

بصورة مثالية، يجب علينا حفط البيانات المقدمة دون نشرها، وإرسال رد في الحال. والتحقق من البيانات التطفلية (spam) يمكن أن يتم خارج النطاق (out of band).

إبراز التعليقات
-----------------------------

.. index::
    single: Command;make:entity

نحتاج إلي تقديم حالة ``state`` للتعليقات مثل: ``submitted``, ``spam``, و ``published``.

أضف خاصية (property) الحالة ``state`` إلي فئة (كلاس) التعليقات ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

وينبغي علينا أن نتأكد من أن الحالة  ``state`` أصبحت ``submitted`` بشكل أساسي (افتراضي default):

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

قم بإنشاء database migration:

.. code-block:: terminal

    $ symfony console make:migration

قم بتعديل ال migration لتحديث جميع التعليقات الموجودة لتصبح ``published`` بشكل تلقائي (default):

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

قم بتحديث قاعدة البيانات (Migrate the database):

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

حدّث منطق العرض لتجنب ظهور التعليقات غير المنشورة في الواجهة الأمامية:

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

قم بتعديل اعدادات ال EasyAdmin لكي تتمكن من رؤية حالة التعليقات:

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

لا تنسَ أيضاً تحديث مصانع الاختبار (test factories): التعليقات التي ينشئها ``CommentFactory`` يجب أن تكون منشورة (published) بشكل افتراضي حتى تظهر في صفحات المؤتمرات (يمكن لأي اختبار تجاوز الحالة عندما يحتاج تعليقاً تحت الإشراف):

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

بالنسبة لاختبارات الموجهات (controller)، قم بمحاكاة التحقق من صحة البيانات (simulate the validation):

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

من اختبارات ال PHPUnit، يمكنك الحصول علي اي خدمة (service) من ال container عن طريق ``self::getContainer()->get()``; كما أنه يمكنك الحصول علي الخدمات الغير عامة (non-public services).

فهم ال Messenger
---------------------

.. index::
    single: Messenger
    single: Components;Messenger

إدارة التعليمات البرمجية (code) الغير متزامنة (asynchronous) في سيمفوني Symfony تعتبر وظيفة مكون ال Messenger:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

عندما ينبغي تنفيذ بعض المنطق (logic) الغير متزامن (asynchronously)، أرسل *رسالة* إلي ناقل الرسول *messenger bus*. يقوم هذا الناقل بتخزين الرسالة في *صف* (قائمة انتظار *queue*) ويعود في الحال ليسمح باستئناف باقي العمليات بأسرع ما يمكن.

يعمل *المستهلك* (*consumer*) بشكل مستمر في الخلفية ليقوم بقرائة الرسائل من قائمة الانتظار (queue) ويقوم بتنفيذ المنطق المرتبط بهذه الرسائل (associated logic). ويمكن للمستهلك أن يعمل علي نفس الخادم (server) الذي يعمل عليه تطبيق الويب أو علي خادم منفصل.

ويشبه جداً الطريقة التي يتم بها معالجة طلبات ال HTTP، بإستثناء أنه ليس لدينا ردود فعل (responses).

برمجة معالج (Handler) الرسالة
----------------------------------------------

لا يجب ان تحتوي الرسالة علي اي منطق (logic) فهي عبارة عن فئة كائن بيانات (data object class). سوف يتم تفكيكها (serialized) ليتم تخزينها في قائمة انتظار، لذلك قم بتخزين بيانات "بسيطة" قابلة للتفكيك (serializable).

قم بإنشاء فئة ال ``CommentMessage``:

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

في عالم الرسول (Messenger)، لا نملك موجهات (متحكمات controllers)، ولكن معالجات (handlers) رسائل.

قم بإنشاء ال `CommentMessageHandler`` تحت مساحة إسم (namespace) جديدة تسمي ``App\MessageHandler`` تعرف كيفية التعامل مع رسائل ال ``CommentMessage``.

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

يساعد ال ``AsMessageHandler`` سيمفوني Symfony علي عمل تسجيل تلقائي (auto-register) والتكوين (الاعداد) التلقائي (auto-configure) للفئة (class) كمعالج للمرسال (Messenger handler). حسب الإتفاق، فإن منطق (logic) المعالج (handle) يعيش في منهج (method) يسمي ``__invoke()``. تلميح (type hint) ال ``CommentMessage`` الموجود علي وسيط هذا المنهج (method) الوحيد يخبر المرسال (Messenger) أي فئة (class) سيقوم بعالجتها.

قم بتعديل الموجهات (المتحكمات controller) لاستخدام النظام الجديد:

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

بدلاً من الإعتماد علي Spam Checker، فإننا الأن نرسل رسالة إلي الناقل (bus). ومن ثم يقرر المعالج ماذا يفعل معها.

لقد حققنا شئ غير متوقع. لقد قمنا بنقل المنطق (logic) لفئة جديدة وفصلنا الموجه (المتحكم controller) عن ال Spam Checker، وهو المعالج (handler). إنها حالة استخدام مثالية للناقل. إختبر الكود، إنه يعمل. كل شئ مازال يتم بشكل متزامن، ولكن الكود ربما يكون *أفضل* بالفعل.

الذهاب المتزامن في الحقيقة
-------------------------------------------------

بشكل تلقائي، يتم النداء علي المعالجات بالتزامن. لاستخدامها بشكل غير متزامن، يجب عليك تهيئة (إعداد) أي صف (قائمة انتظار) لكل معالج في  ملف الاعدادات الموجود في ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

ملف الاعدادت يخبر الناقل أن يرسل حالات (أمثلة instances) من ``App\Message\CommentMessage`` الي صف ال ``async``، المُعرَّف عن طريق DSN (``MESSENGER_TRANSPORT_DSN``)، والذي يشير إلي Doctrine كما هو مُعَدّ في ``.env``. بعبارة أبسط، نحن نستخدم PostgreSQL كصف (قائمة انتظار) لرسائلنا.

.. tip::

    وراء الكواليس، تستخدم سيمفوني نظام النشر/النشر الفرعي المدمج والأداء PostgreSQL والقابل للتطوير والمعاملات الخاصة ب( ``LISTEN`` / ``NOTIFY``). يمكنك أيضًا قراءة فصل RabbitMQ إذا كنت تريد استخدامه بدلاً من PostgreSQL كوسيط للرسائل.

إستهلاك الرسائل
-----------------------------

إذا حاولت إرسال تعليق جديد، لن يتم إستدعاء (إستخدام) محقق البيانات الفضولية (العشوائية spam checker) بعد الان. قم بإضافة ``error_log()`` في منهج (method) ال ``getSpamScore()`` للتأكيد. بدلاً من ذلك، هناك رسالة منتظرة في الانتظار queue، مستعدة ليتم استهلاكها (مناداتها) عن طريق بعض العمليات.

.. index::
    single: Command;messenger:consume

كما قد تخيلت، فإن سيمفوني يأتي معه أمر خاص بالمستهلك. قم بإستخدامه الأن:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

سيقوم علي الفور بإستهلاك الرسالة المرسلة الخاصة بالتعليق الذي تم إرساله:

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

عملية إستهلاك الرسالة تم تسجيلها، ولكن تحصل علي رد فوري علي وحدة التحكم (الكونسل) عن طريق تمرير أمر ال ``-vv``. يمكنك حتي تحديد المناداة ل OpenAI API.

لإقاف المستهلك، قم بضغط ``Ctrl+C``.

تشغيل العمال (Workers) في الخلفية
-----------------------------------------------------

بدلا من إطلاق المستهلك كل مرة نقوم بإضافة تعليق وإقافه فورا بعد ذلك، نريد ان نجعله يعمل بشكل مستمر بدون الحاجة الي العديد من شاشات التيرمنل او تبويبات (tabs) مفتوحة.

السيمفوني CLI يمكن ادارة مثل هذه الاوامر في الخلفية عن طريق اضافة علامة الشيطان (daemon flag) الي الامر عند التشغيل ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

قم بتشغيل مستهلك الرسالة مرة اخري، ولكن ارسله الي الخلفية:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

خيار ``--watch`` يخبر سيمفوني بانه عليه ان يقوم بإعادة تشغيل الامر كلما كان هناك تغير في اي ملف يقع تحت ال ``config/``، ``src/``، ``templates/``، او ``vendor/``.

.. note::

    لا تقم باستخدام ``-vv`` لانه سوف تجد رسائل متكررة في ``server:log`` (الرسائل المسجلة ورسائل وحدة التحكم (الكونسل)).

اذا توقف المستهلك عن العمل لاي سبب (حدود الذاكرة، مشكلة، ...)، سيتم إعادة تشغيله تلقائياً. ولو فشل المستهلك بسرعة أكبر مما يجب، سيمفوني CLI سوف يستسلم.

.. index::
    single: Symfony CLI;server:log

سوف تتدفق السجلات عن طريق ``symfony server:log`` مع جميع السجلات القادمة من PHP، خادم الويب، والتطبيق (الابلاكيشن):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

استخدم أمر ال ``server:status`` لسرد جميع العمال بالخلفية الذي يتم ادارتهم عن طريق المشروع الحالي:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

لإقاف احد العمال، قم باقاف خادم الويب او اقتل ال PID المعطي عن طريق امر ال ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

إعادة محاولة الرسائل الفاشلة
-----------------------------------------------------

ماذا  لو ان قاعدة البيانات سقطت (توقفت) أثناء استهلاك رسالة؟ لا يوجد اي تأثير علي الاشخاص الذين يقدمون (يرسلون) التعليقات، لكن الرسالة تفشل ولم يتم التحقق من البيانات الفضولية.

الرسول (ماسنجر Messenger) يحتوي علي آلية إعادة المحاولة عندم تحدث مشكلة أثناء معالجة الرسالة:

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

لو أن مشكلة حدثت أثناء معالجة الرسالة، المستهلك سيقوم بإعادة المحاولة ثلاث مرات قبل أن يستسلم. ولكن بدلاً من تجاهل الرسالة، سوف يقوم بتخزينها بشكل دائم في قائمة انتظار ال ``failed``، التي تستخدم جدول مختلف من قاعدة البيانات.

إفحص الرسائل الفاشلة وقم باعادة محاولة إرسالهم عن طريق الاوامر التالية:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

تشغيل العمال علي Upsun
-------------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

لإستهلاك الرسائل من PostgreSQL، نحتاج الي تشغيل امر ال ``messenger:consume`` بشكل مستمر. علي Upsun، هذا هو دور *العامل*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

مثل سيمفوني CLI، يقوم Upsun بادارة عمليات إعادة التشغيل والسجلات.

.. index::
    single: Symfony CLI;cloud:logs

للحصول علي سجلات العامل، قم بأستخدام:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:logs --worker=messages all

.. sidebar:: الذهاب أبعد من ذلك

    * الدروس التعليمية الخاصة بالرسول:  `SymfonyCasts Messenger tutorial`_؛

    * تصميم ناقل الخدمات المؤسسة ووتيرة CQRS: `Enterprise service bus`_  و `CQRS pattern`_؛

    * مراجع رسول سيمفوني: `Symfony Messenger docs`_؛

.. _`SymfonyCasts Messenger tutorial`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS pattern`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony Messenger docs`: https://symfony.com/doc/current/messenger.html
