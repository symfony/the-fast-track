اطلاع‌رسانی با تمام قوا
============================================

اپلیکیشن Guestbook، بازخوردهای مربوط به کنفرانس‌ها را جمع‌آوری می‌کند. اما ما در بازخورددادن به کاربرانمان عالی نیستیم.

از آنجایی که کامنت‌ها تعدیل می‌شوند، کاربران احتمالاً نمی‌فهمند چرا کامنت‌هایشان بی‌درنگ منتشر نمی‌شود. آن‌ها حتی ممکن است با تصور اینکه مشکلی فنی پیش آمده، کامنت‌هایشان را دوباره ارسال کنند. دادن بازخورد به آن‌ها پس از ارسال یک کامنت عالی خواهد بود.

علاوه بر این، احتمالاً باید وقتی کامنت کاربران منتشر شد به آن‌ها اطلاع دهیم. ما از آن‌ها رایانامه درخواست می‌کنیم تا بتوانیم از آن استفاده کنیم.

راه‌های زیادی برای اطلاع‌رسانی به کاربران وجود دارد. رایانامه اولین گزینه‌ای است که ممکن است به آن فکر کنید، اما اعلان‌ها (notifications) در اپلیکیشن وب نیز یک گزینه‌ی دیگر است. حتی ما می‌توانیم به ارسال پیامک (SMS) و یا ارسال یک پیغام به Slack یا Telegram فکر کنیم. گزینه‌های زیادی وجود دارد.

.. index::
    single: Components;Notifier
    single: Notifier

کامپوننت اعلانگر (Notifier) سیمفونی تعداد زیادی راهبرد اطلاع‌رسانی را پیاده‌سازی می‌کند.

ارسال اعلان‌های اپلیکیشن وب در مرورگر
----------------------------------------------------------------------

.. index::
    single: Flash Messages

به عنوان اولین گام، بیایید در مرورگر پس از ارسال کامنت، به کاربران اعلام کنیم که کامنت‌ها تعدیل می‌شوند:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -16,6 +16,8 @@ use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\Notification\Notification;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -45,7 +47,8 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        NotifierInterface $notifier,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -69,8 +72,14 @@ final class ConferenceController extends AbstractController
                 ];
                 $this->bus->dispatch(new CommentMessage($comment->getId(), $context));

    +            $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));
    +
                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);
             }

    +        if ($form->isSubmitted()) {
    +            $notifier->send(new Notification('Can you check your submission? There are some problems with it.', ['browser']));
    +        }
    +
             $paginator = $commentRepository->getCommentPaginator($conference, $offset);

اعلان‌گر، *اعلان (notification)* را از طریق یک *کانال (channel)* به *گیرندگان (recipients)* *ارسال می‌کند*.

اعلان دارای یک عنوان، یک محتوای اختیاری و یک درجه‌ی اهمیت است.

یک اعلان با توجه به اهمیتش، به یک یا چند کانال ارسال می‌گردد. شما می‌توانید پیغام‌های ضروری و فوری را از طریق پیامک و پیغام‌های معمولی را از طریق رایانامه ارسال کنید.

ما برای اعلان‌های مرورگر، گیرنده نداریم.

.. index::
    single: Twig;for

اعلان‌های مرورگر، از طریق بخش *اعلان*، از *پیغام‌های flash* استفاده می‌کند. ما نیاز داریم که آن‌ها را با به‌روزرسانی قالب کنفرانس، نمایش دهیم:

.. code-block:: diff
    :caption: patch_file

    --- i/templates/conference/show.html.twig
    +++ w/templates/conference/show.html.twig
    @@ -3,6 +3,13 @@
     {% block title %}Conference Guestbook - {{ conference }}{% endblock %}

     {% block body %}
    +    {% for message in app.flashes('notification') %}
    +        <div class="alert alert-info alert-dismissible fade show">
    +            {{ message }}
    +            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    +        </div>
    +    {% endfor %}
    +
         <h2 class="mb-5">
             {{ conference }} Conference
         </h2>

حالا به کاربران اطلاع داده می‌شود که کامنت‌های ارسالی‌شان تعدیل می‌گردد:

.. figure:: screenshots/form-success-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

به عنوان یک دست‌آورد جانبی، اگر خطایی در فرم وجود داشته باشد، ما یک پیغام اعلان زیبا در بالای وب‌سایت دریافت می‌کنیم:

.. figure:: screenshots/form-error-notification.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

.. tip::

    پیغام‌های Flash، از سیستم *نشست HTTP* به عنوان انبار میانی استفاده می‌کنند. اثر اصلی این مسئله این است که نهان‌سازی HTTP در غیرفعال می‌شود چرا که سیستم نشست (session) باید شروع شده باشد تا بتوانیم ببینیم پیغام یا پیغام‌هایی وجود دارد یا خیر.

    این دلیل آن است که ما قطعه‌ی مربوط به پیغام‌های flash را در قالب ``show.html.twig`` اضافه کردیم و آن را در قالب پایه قرار ندادیم. چرا که اگر این کار را می‌کردیم، نهان‌سازی HTTP را در صفحه‌ی اصلی از دست می‌دادیم.

اطلاع‌رسانی به مدیران از طریق رایانامه
------------------------------------------------------------------------

به جای ارسال رایانامه از طریق ``MailerInterface`` برای اطلاع‌رسانی به مدیر در مورد ارسال‌شدن یک کامنت، در رسیدگی‌کننده‌ی پیغام از کامپوننت اعلانگر استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -4,15 +4,15 @@ namespace App\MessageHandler;

     use App\ImageOptimizer;
     use App\Message\CommentMessage;
    +use App\Notification\CommentReviewNotification;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    -use Symfony\Bridge\Twig\Mime\NotificationEmail;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    -use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
     use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
    @@ -24,8 +24,7 @@ class CommentMessageHandler
             private CommentRepository $commentRepository,
             private MessageBusInterface $bus,
             private WorkflowInterface $commentStateMachine,
    -        private MailerInterface $mailer,
    -        #[Autowire('%admin_email%')] private string $adminEmail,
    +        private NotifierInterface $notifier,
             private ImageOptimizer $imageOptimizer,
             #[Autowire('%photo_dir%')] private string $photoDir,
             private ?LoggerInterface $logger = null,
    @@ -50,13 +49,7 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    -            $this->mailer->send((new NotificationEmail())
    -                ->subject('New comment posted')
    -                ->htmlTemplate('emails/comment_notification.html.twig')
    -                ->from($this->adminEmail)
    -                ->to($this->adminEmail)
    -                ->context(['comment' => $comment])
    -            );
    +            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
             } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

متد ``getAdminRecipients()``، مدیران گیرنده را همانطور که در پیکربندی اعلانگر پیکربندی شده است، بازمی‌گرداند؛ آن را به‌روزرسانی کرده و آدرس رایانامه‌تان را به آن بیافزایید:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/notifier.yaml
    +++ w/config/packages/notifier.yaml
    @@ -9,4 +9,4 @@ framework:
                 medium: ['email']
                 low: ['email']
             admin_recipients:
    -            - { email: admin@example.com }
    +            - { email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%" }

حالا کلاس ``CommentReviewNotification`` را ایجاد کنید:

.. code-block:: php
    :caption: src/Notification/CommentReviewNotification.php

    namespace App\Notification;

    use App\Entity\Comment;
    use Symfony\Component\Notifier\Message\EmailMessage;
    use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
    use Symfony\Component\Notifier\Notification\Notification;
    use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;

    class CommentReviewNotification extends Notification implements EmailNotificationInterface
    {
        public function __construct(
            private Comment $comment,
        ) {
            parent::__construct('New comment posted');
        }

        public function asEmailMessage(EmailRecipientInterface $recipient, string $transport = null): ?EmailMessage
        {
            $message = EmailMessage::fromNotification($this, $recipient, $transport);
            $message->getMessage()
                ->htmlTemplate('emails/comment_notification.html.twig')
                ->context(['comment' => $this->comment])
            ;

            return $message;
        }
    }

متد ``asEmailMessage()`` در رابط ``EmailNotificationInterface`` اختیاری است، اما این امکان را می‌دهد تا رایانامه را سفارشی‌سازی کنید.

یک مزیت استفاده از اعلانگر به جای ارسال مستقیم رایانامه از طریق mailer این است که اعلان را از «کانال» مورد استفاده برای آن مجزا می‌کند. همانطور که می‌توانید ببینید، هیچ چیزی صریحاً نمی‌گوید که اعلان باید از طریق رایانامه ارسال گردد.

در عوض، کانال در ``config/packages/notifier.yaml`` بر اساس *درجه‌ی اهمیت* اعلان (به صورت پیش‌فرض ``low``)، پیکربندی شده است:

.. code-block:: yaml
    :caption: config/packages/notifier.yaml
    :class: ignore

    framework:
    notifier:
        channel_policy:
            # use chat/slack, chat/telegram, sms/twilio or sms/nexmo
            urgent: ['email']
            high: ['email']
            medium: ['email']
            low: ['email']

ما در مورد کانال‌های ``browser`` و ``email`` صحبت کردیم. بیایید چند کانال جالب‌تر ببینیم.

چت‌کردن با مدیران
---------------------------------

.. index::
    single: Slack

بیایید صادق باشیم، ما منتظر بازخوردهای مثبت هستیم. یا حداقل بازخوردی سازنده. اگر کسی کامنتی حاوی کلماتی همچون «عالی» یا «فوق‌العاده» ارسال کند، ما ممکن است بخواهیم آن را زودتر از سایر کامنت‌ها بپذیریم.

ما می‌خواهیم برای چنین پیغام‌هایی، علاوه بر رایانامه‌های همیشگی، بر روی سیستم‌های پیغام‌رسانی آنی مثل Slack یا Telegram، هشدار دریافت کنیم.

.. index::
    single: Components;Notifier
    single: Notifier

پشتیبانی از Slack را بر روی اعلانگر سیمفونی نصب کنید:

.. code-block:: terminal

    $ symfony composer req slack-notifier

برای شروع، DSN مربوط به Slack را با یک توکن دسترسی Slack و شناسه‌ی کانال Slack‌ای که می‌خواهید به آن پیغام‌ها را ارسال نمایید، بسازید: ``slack://ACCESS_TOKEN@default?channel=CHANNEL``.

.. index::
    single: Command;secrets:set

از آنجایی که توکن دسترسی، حساس است، DSN مربوط به Slack را در انبار رمز ذخیره کنید:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN

همین کار را برای محیط عمل‌آوری نیز انجام دهید:

.. code-block:: terminal
    :class: answers(slack://ACCESS_TOKEN@default?channel=CHANNEL)

    $ symfony console secrets:set SLACK_DSN --env=prod

کلاس اعلان را به‌روزرسانی کنید تا پیغام‌ها را با توجه به محتوای متن آن‌ها راه‌یابی کند (یک regex ساده این کار را انجام می‌دهد):

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -7,6 +7,7 @@ use Symfony\Component\Notifier\Message\EmailMessage;
     use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;
    +use Symfony\Component\Notifier\Recipient\RecipientInterface;

     class CommentReviewNotification extends Notification implements EmailNotificationInterface
     {
    @@ -26,4 +27,15 @@ class CommentReviewNotification extends Notification implements EmailNotificatio

             return $message;
         }
    +
    +    public function getChannels(RecipientInterface $recipient): array
    +    {
    +        if (preg_match('{\b(great|awesome)\b}i', $this->comment->getText())) {
    +            return ['email', 'chat/slack'];
    +        }
    +
    +        $this->importance(Notification::IMPORTANCE_LOW);
    +
    +        return ['email'];
    +    }
     }

ما همچنین درجه‌ی اهمیت کامنت‌های «معمولی» را تغییر دادیم تا کمی طراحی رایانامه را اصلاح کنیم.

و تمام! یک کامنت که در متن آن «awesome» وجود دارد را ارسال کنید، شما باید یک پیغام در Slack دریافت کنید.

همچون رایانامه‌ها، اینجا هم می‌توانید ``ChatNotificationInterface`` را پیاده‌سازی کنید تا نحوه‌ی renderشدن پیش‌فرض پیغام Slack را بازنویسی کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -3,13 +3,18 @@
     namespace App\Notification;

     use App\Entity\Comment;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
    +use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
    +use Symfony\Component\Notifier\Message\ChatMessage;
     use Symfony\Component\Notifier\Message\EmailMessage;
    +use Symfony\Component\Notifier\Notification\ChatNotificationInterface;
     use Symfony\Component\Notifier\Notification\EmailNotificationInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;
     use Symfony\Component\Notifier\Recipient\RecipientInterface;

    -class CommentReviewNotification extends Notification implements EmailNotificationInterface
    +class CommentReviewNotification extends Notification implements EmailNotificationInterface, ChatNotificationInterface
     {
         public function __construct(
             private Comment $comment,
    @@ -28,6 +33,28 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
             return $message;
         }

    +    public function asChatMessage(RecipientInterface $recipient, string $transport = null): ?ChatMessage
    +    {
    +        if ('slack' !== $transport) {
    +            return null;
    +        }
    +
    +        $message = ChatMessage::fromNotification($this, $recipient, $transport);
    +        $message->subject($this->getSubject());
    +        $message->options((new SlackOptions())
    +            ->iconEmoji('tada')
    +            ->iconUrl('https://guestbook.example.com')
    +            ->username('Guestbook')
    +            ->block((new SlackSectionBlock())->text($this->getSubject()))
    +            ->block(new SlackDividerBlock())
    +            ->block((new SlackSectionBlock())
    +                ->text(sprintf('%s (%s) says: %s', $this->comment->getAuthor(), $this->comment->getEmail(), $this->comment->getText()))
    +            )
    +        );
    +
    +        return $message;
    +    }
    +
         public function getChannels(RecipientInterface $recipient): array
         {
             if (preg_match('{\b(great|awesome)\b}i', $this->comment->getText())) {

این بهتر است، اما بیایید یک گام جلوتر برویم. فوق‌العاده نبود اگر قادر بودیم که کامنت را مستقیماً از درون Slack تأیید یا رد کنیم؟

اعلان را تغییر دهیم تا URL بازبینی را بپذیرد و ۲ دکمه در پیغام Slack اضافه کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Notification/CommentReviewNotification.php
    +++ w/src/Notification/CommentReviewNotification.php
    @@ -3,6 +3,7 @@
     namespace App\Notification;

     use App\Entity\Comment;
    +use Symfony\Component\Notifier\Bridge\Slack\Block\SlackActionsBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
     use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
     use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
    @@ -18,6 +19,7 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
     {
         public function __construct(
             private Comment $comment,
    +        private string $reviewUrl,
         ) {
             parent::__construct('New comment posted');
         }
    @@ -50,6 +52,10 @@ class CommentReviewNotification extends Notification implements EmailNotificatio
                 ->block((new SlackSectionBlock())
                     ->text(sprintf('%s (%s) says: %s', $this->comment->getAuthor(), $this->comment->getEmail(), $this->comment->getText()))
                 )
    +            ->block((new SlackActionsBlock())
    +                ->button('Accept', $this->reviewUrl, 'primary')
    +                ->button('Reject', $this->reviewUrl.'?reject=1', 'danger')
    +            )
             );

             return $message;

حالا نوبت دنبال کردن تغییرات به صورت عقب‌رو است؛ ابتدا رسیدگی‌کننده‌ی پیغام را به‌روزرسانی کنید تا URL بازبینی را در اختیار داشته باشد:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -49,7 +49,8 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    -            $this->notifier->send(new CommentReviewNotification($comment), ...$this->notifier->getAdminRecipients());
    +            $notification = new CommentReviewNotification($comment, $message->getReviewUrl());
    +            $this->notifier->send($notification, ...$this->notifier->getAdminRecipients());
             } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
                 if ($comment->getPhotoFilename()) {
                     $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());

همانطور که می‌توانید ببینید، URL بازبینی باید بخشی از پیغام کامنت باشد، حالا بیایید آن را اضافه کنیم:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Message/CommentMessage.php
    +++ w/src/Message/CommentMessage.php
    @@ -6,10 +6,16 @@ class CommentMessage
     {
         public function __construct(
             private int $id,
    +        private string $reviewUrl,
             private array $context = [],
         ) {
         }

    +    public function getReviewUrl(): string
    +    {
    +        return $this->reviewUrl;
    +    }
    +
         public function getId(): int
         {
             return $this->id;

در نهایت، کنترلرها را به‌روزرسانی کنید تا URL بازبینی را تولید کرده و آن را به سازنده‌ی پیغام کامنت بدهند:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -12,6 +12,7 @@ use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
     use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    @@ -42,7 +43,8 @@ class AdminController extends AbstractController
             $this->entityManager->flush();

             if ($accepted) {
    -            $this->bus->dispatch(new CommentMessage($comment->getId()));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl));
             }

             return new Response($this->twig->render('admin/review.html.twig', [
    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -17,6 +17,7 @@ use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Notifier\Notification\Notification;
     use Symfony\Component\Notifier\NotifierInterface;
     use Symfony\Component\Routing\Attribute\Route;
    +use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

     final class ConferenceController extends AbstractController
     {
    @@ -70,7 +71,8 @@ final class ConferenceController extends AbstractController
                     'referrer' => $request->headers->get('referer'),
                     'permalink' => $request->getUri(),
                 ];
    -            $this->bus->dispatch(new CommentMessage($comment->getId(), $context));
    +            $reviewUrl = $this->generateUrl('review_comment', ['id' => $comment->getId()], UrlGeneratorInterface::ABSOLUTE_URL);
    +            $this->bus->dispatch(new CommentMessage($comment->getId(), $reviewUrl, $context));

                 $notifier->send(new Notification('Thank you for the feedback; your comment will be posted after moderation.', ['browser']));

جداسازی (decoupling) کد باعث لزوم انجام تغییرات در جاهای بیشتری است، اما آزمودن، دلیل‌‌آوری و بازاستفاده را آسان‌تر می‌کند.

مجدداً تلاش کنید، حالا باید پیغام شکل مناسبی داشته باشد:

.. image:: images/slack-message.png
    :align: center

ناهمزمان‌کردن تمام موارد
-----------------------------------------------

اعلان‌ها مانند رایانامه‌ها به صورت پیش‌فرض به شکل ناهمزمان ارسال می‌شوند:

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 5,6
    :class: ignore

    framework:
        messenger:
            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async

                # Route your messages to the transports
                App\Message\CommentMessage: async

اگر می‌خواستیم پیغام‌های ناهمزمان را غیرفعال کنیم، با یک مشکل جزئی روبرو می‌شدیم. برای هر کامنت، ما یک رایانامه و یک پیغام Slack دریافت می‌کنیم. اگر خطایی در پیغام Slack رخ دهد (شناسه کانال اشتباه، توکن غلط و ...)، پیغام‌رسان قبل از دورانداختن پیغام، ۳ بار بازتلاش انجام می‌دهد. اما از آنجایی که رایانامه در همان بار اول ارسال شده است، ما ۳ بار رایانامه دریافت می‌کنیم و هیچ پیغام Slackای دریافت نخواهیم کرد.

از آنجایی که همه‌چیز ناهمزمان است، پیغام‌ها از هم مستقل می‌شوند. پیغام‌های پیامک نیز از پیش به صورت ناهمزمان پیکربندی شده‌اند که اگر خواستید بر روی تلفن‌همراه‌تان هم اطلاع‌رسانی شوید.

اطلاع‌رسانی به کاربران از طریق رایانامه
--------------------------------------------------------------------------

آخرین وظیفه اطلاع‌رسانی به کاربران در هنگامی است که کامنت‌شان تأیید گردیده است. چطور است که پیاده‌سازی این مورد به خودتان واگذار شود؟

.. sidebar:: بیشتر بدانید

    * `پیغام‌های flash در سیمفونی`_.

.. _`پیغام‌های flash در سیمفونی`: https://symfony.com/doc/current/controller.html#flash-messages
