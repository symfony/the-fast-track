مراسلة المدراء
===========================

.. index::
    single: Components;Mailer
    single: Mailer
    single: Emails

لضمان تعليقات عالية الجودة ، يجب على المشرف الإشراف على جميع التعليقات. عندما يكون التعليق في حالة ``ham`` أو ``potential_spam`` ، يجب إرسال *بريد إلكتروني* إلى المسؤول برابطين: أحدهما لقبول التعليق والآخر لرفضه.

اعداد بريد خاص بالمدير
-----------------------------------------

لحفظ البريد الالكتروني للمدير، إستخدم مُعامل حاوية. لغرض العرض، نسمح أيضاً ان يتم وضعه عن طريق مُتغير بيئة العمل (لا يجب ان تكون هناك حاجة لهذا في الحياة الحقيقية):

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -5,6 +5,8 @@
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
         photo_dir: "%kernel.project_dir%/public/uploads/photos"
    +    default_admin_email: admin@example.com
    +    admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"

     services:
         # default configuration for services in *this* file

قد تتم مُعالجة متغير بيئة العمل قبل إستخدامه. هنا نستخدم المُعالج الإفتراضي (``default``) للعودة الي قيمة مُعامِل الـ ``default_admin_email`` لو ان متغير بيئة العمل ``ADMIN_EMAIL`` غير موجود.

إرسال رسالة إعلام بالبريد الإلكتروني
--------------------------------------------------------------------

لإرسال بريد إلكتروني، يمكنك الاختيار بين عدة مُلخصات (abstractions) لفئة الـ ``Email``؛ من الـ ``Message`` أدني مستوي الي الـ  ``NotificationEmail`` أعلي مستوي. في الأغلب سوف تستخدم فئة الـ ``Email`` أكثر، ولكن ``NotificationEmail`` أفضل إختيار لرسائل البريد الإلكترونية الداخلية.

في معالج الرسائل ، دعنا نعوض منطق التحقق-الالي

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -7,6 +7,9 @@ use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    +use Symfony\Bridge\Twig\Mime\NotificationEmail;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
    +use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -20,6 +23,8 @@ class CommentMessageHandler
             private CommentRepository $commentRepository,
             private MessageBusInterface $bus,
             private WorkflowInterface $commentStateMachine,
    +        private MailerInterface $mailer,
    +        #[Autowire('%admin_email%')] private string $adminEmail,
             private ?LoggerInterface $logger = null,
         ) {
         }
    @@ -42,8 +47,13 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    -            $this->commentStateMachine->apply($comment, $this->commentStateMachine->can($comment, 'publish') ? 'publish' : 'publish_ham');
    -            $this->entityManager->flush();
    +            $this->mailer->send((new NotificationEmail())
    +                ->subject('New comment posted')
    +                ->htmlTemplate('emails/comment_notification.html.twig')
    +                ->from($this->adminEmail)
    +                ->to($this->adminEmail)
    +                ->context(['comment' => $comment])
    +            );
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

تعد ``MailerInterface`` نقطة الدخول الرئيسية وتسمح بإرسال``() send `` البريد الإلكتروني.

لإرسال بريد إلكتروني، نحتاج الي مُرسِل (عنوان الـ مِن/مُرسِل ``From``/``Sender``). بدلاً من تعريفه بشكل صريح علي نموذج (instance) البريد الالكتروني، عرفها بشكل عام:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/mailer.yaml
    +++ w/config/packages/mailer.yaml
    @@ -1,3 +1,5 @@
     framework:
         mailer:
             dsn: '%env(MAILER_DSN)%'
    +        envelope:
    +            sender: "%admin_email%"

تمديد قالب البريد الإلكتروني للاشعار
--------------------------------------------------------------------

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;url

يرث قالب الإشعار بالبريد الإلكتروني من قالب البريد الإلكتروني للإشعار الافتراضي الذي يأتي مع سيمفوني:

.. code-block:: html+twig
    :caption: templates/emails/comment_notification.html.twig

    {% extends '@email/default/notification/body.html.twig' %}

    {% block content %}
        Author: {{ comment.author }}<br />
        Email: {{ comment.email }}<br />
        State: {{ comment.state }}<br />

        <p>
            {{ comment.text }}
        </p>
    {% endblock %}

    {% block action %}
        <spacer size="16"></spacer>
        <button href="{{ url('review_comment', { id: comment.id }) }}">Accept</button>
        <button href="{{ url('review_comment', { id: comment.id, reject: true }) }}">Reject</button>
    {% endblock %}

يقوم القالب بتجاوز بعض الكتل لتخصيص رسالة البريد الالكتروني وإضافة بعض الروابط التي تمكن المسؤول من قبول او رفض التعليق. تتم إضافة وسيطة مسار (route argument) ليست مُعامل مسار صالحة كعنصر جملة الاستعلام (query string item) (رابط الرفض يبدو مثل ``/admin/comment/review/42?reject=true``).

قالب الـ ``NotificationEmail`` الاساسي (الافتراضي) يستخدم `Inky`_  بدلاً من HTML لتصميم الرسائل الالكترونية. تساعد علي إنشاء بريد إلكتروني سريعة الاستجابة (responsive) تتوافق مع جميع عملاء البريد الالكتروني الاكثر رواجاً.

لتحقيق أقصى قدر من التوافق مع برامج قراءة البريد الإلكتروني ، يشتمل تخطيط قاعدة الإعلام على جميع أوراق الأنماط (عبر حزمة CSS inliner) بشكل افتراضي.

تعتبر هاتان الميزتان جزءًا من ملحقات Twig الاختيارية التي يجب تثبيتها:

.. code-block:: terminal

    $ symfony composer req "twig/cssinliner-extra:^3" "twig/inky-extra:^3"

توليد عناوين URL المطلقة في أمر
------------------------------------------------------

.. index::
    single: Twig;Link
    single: Link

في رسائل البريد الالكترونية، إنشاء روابط باستخدام ``url()`` بدلاً من ``path()`` لانك تحتاج روابط كاملة (مع المُخطط والمُضيف).

يتم إرسال البريد الإلكتروني من معالج الرسائل ، في سياق وحدة التحكم. يعد إنشاء عناوين URL المطلقة في سياق الويب أسهل حيث نعرف مخطط ونطاق الصفحة الحالية. ليس هذا هو الحال في سياق وحدة التحكم.

حدد اسم النطاق ومخطط الاستخدام بشكل صريح:

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -7,6 +7,7 @@ parameters:
         photo_dir: "%kernel.project_dir%/public/uploads/photos"
         default_admin_email: admin@example.com
         admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"
    +    default_base_url: 'http://127.0.0.1'

     services:
         # default configuration for services in *this* file

ثم اطلب من المُوجِّه (router) استخدامه كعنوان URI الافتراضي عند توليد الروابط خارج سياق طلب HTTP:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/routing.yaml
    +++ w/config/packages/routing.yaml
    @@ -3,3 +3,3 @@ framework:
             # Configure how to generate URLs in non-HTTP contexts, such as CLI commands.
             # See https://symfony.com/doc/current/routing.html#generating-urls-in-commands
    -        default_uri: '%env(DEFAULT_URI)%'
    +        default_uri: '%env(default:default_base_url:SYMFONY_DEFAULT_ROUTE_URL)%'

يتم تعريف متغير بيئة العمل ``SYMFONY_DEFAULT_ROUTE_URL`` محلياً تلقائياً عند استخدام شاشة اوامر سيمفوني (symfony CLI) ويتم تحديده بناءاً علي الاعدادات علي Upsun.

توصيل مسار (Route) إلى جهاز تحكم (Controller)
-----------------------------------------------------------------

مسار ``review_comment`` غير موجود حتى الآن ، فلننشئ وحدة تحكم المدير للتعامل معه:

.. code-block:: php
    :caption: src/Controller/AdminController.php

    namespace App\Controller;

    use App\Entity\Comment;
    use App\Message\CommentMessage;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Messenger\MessageBusInterface;
    use Symfony\Component\Routing\Attribute\Route;
    use Symfony\Component\Workflow\WorkflowInterface;
    use Twig\Environment;

    class AdminController extends AbstractController
    {
        public function __construct(
            private Environment $twig,
            private EntityManagerInterface $entityManager,
            private MessageBusInterface $bus,
        ) {
        }

        #[Route('/admin/comment/review/{id}', name: 'review_comment')]
        public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
        {
            $accepted = !$request->query->get('reject');

            if ($commentStateMachine->can($comment, 'publish')) {
                $transition = $accepted ? 'publish' : 'reject';
            } elseif ($commentStateMachine->can($comment, 'publish_ham')) {
                $transition = $accepted ? 'publish_ham' : 'reject_ham';
            } else {
                return new Response('Comment already reviewed or not in the right state.');
            }

            $commentStateMachine->apply($comment, $transition);
            $this->entityManager->flush();

            if ($accepted) {
                $this->bus->dispatch(new CommentMessage($comment->getId()));
            }

            return new Response($this->twig->render('admin/review.html.twig', [
                'transition' => $transition,
                'comment' => $comment,
            ]));
        }
    }

يبدا رابط مراجعة التعليق بـ ``/admin/`` لحمايته بواسطة جدار الحماية المُحدد في الخطوة السابقة. يجب مُصادقة (authenticated) المسؤول للوصول الي هذا المصدر (resource).

بدلاص من إنشاء نموذج رد (``Response``)، قمنا باستخدام ``render()``، دالة إختصار مُقدمة بواسطة الفئة الاساسية لوحدة تحكم الـ ``AbstractController``.

.. index::
    single: Twig;extends
    single: Twig;block

عند الانتهاء من المراجعة ، يشكر قالب قصير المشرف على عملهم الشاق:

.. code-block:: html+twig
    :caption: templates/admin/review.html.twig

    {% extends 'base.html.twig' %}

    {% block body %}
        <h2>Comment reviewed, thank you!</h2>

        <p>Applied transition: <strong>{{ transition }}</strong></p>
        <p>New state: <strong>{{ comment.state }}</strong></p>
    {% endblock %}

باستخدام بريد الماسك (Mail Catcher)
-----------------------------------------------------

.. index::
    single: Docker;Mail Catcher

بدلاً من استخدام خادم SMTP "حقيقي" أو موفر جهة خارجية لإرسال رسائل البريد الإلكتروني ، دعنا نستخدم أداة التقاط البريد (Mail catcher). يوفر أداة التقاط البريد خادم SMTP لا يقوم بتسليم رسائل البريد الإلكتروني ، ولكنه يجعلها متاحة من خلال واجهة ويب بدلاً من ذلك. لحسن الحظ، قام Symfony بإعداد ملتقط البريد هذا تلقائيًا لنا:

.. code-block:: yaml
    :caption: compose.override.yaml
    :class: ignore

    ###> symfony/mailer ###
    mailer:
        image: axllent/mailpit
        ports:
        - "1025"
        - "8025"
        environment:
        MP_SMTP_AUTH_ACCEPT_ANY: 1
        MP_SMTP_AUTH_ALLOW_INSECURE: 1
    ###< symfony/mailer ###

الوصول إلى بريد الويب (webmail)
-------------------------------------------------

.. index::
    single: Symfony CLI;open:local:webmail

يمكنك فتح بريد الويب (webmail) من محطة طرفية (terminal):

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:webmail

أو من شريط أدوات تصحيح الويب (web debug toolbar):

.. figure:: screenshots/webmail-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

إرسال تعليق ، يجب أن تتلقى بريدًا إلكترونيًا في واجهة بريد الويب (webmail):

.. figure:: screenshots/webmail.png
    :alt: /
    :align: center
    :figclass: with-browser

انقر على عنوان البريد الإلكتروني على الواجهة واقبل أو ارفض التعليق كما تراه مناسبًا:

.. figure:: screenshots/webmail-rejected.png
    :alt: /
    :align: center
    :figclass: with-browser

تحقق من السجلات عن طريق ``server:log`` إذا لم يعمل ذلك كما هو متوقع.

إدارة البرامج النصية طويلة الأمد
------------------------------------------------------------

يأتي وجود نصوص طويلة الأمد بسلوكيات يجب أن تكون على دراية بها.على عكس نموذج PHP المستخدم لـ HTTP حيث يبدأ كل طلب بحالة نظيفة ، فإن مستهلك الرسالة يعمل بشكل مستمر في الخلفية.كل معالجة للرسالة ترث الحالة الحالية ، بما في ذلك ذاكرة التخزين المؤقت. لتجنب أي مشاكل في العقيدة ، يتم مسح مديري الكيانات تلقائيًا بعد معالجة الرسالة.يجب عليك التحقق مما إذا كانت خدماتك الخاصة تحتاج إلى القيام بنفس الشيء أم لا.

إرسال رسائل البريد الإلكتروني بشكل غير متزامن
------------------------------------------------------------------------------------

قد يستغرق إرسال البريد الالكتروني المُرسل في مُعالج الرسائل بعض الوقت. بل ربما تُلقي إعتراض (exception). في حالة ان اعتراض حدث اثناء معالجة الرسالة، ستتم إعادةالمحاولة. لكن بدلاً من إعادة محاولة استهلاك رسالة التعليق، سوف يكون من الافضل إعادة محاولة إرسال البريد الالكتروني.

نحن نعرف بالفعل كيفية القيام بذلك: أرسل رسالة البريد الإلكتروني في الحافلة.

نموذج ``MailerInterface`` يقوم بالعمل الشاق: عند تعريف ناقل، يُرسل رسائل البريد الالكتروني إليه بدلاً من إرسالهم. لا حاجة لاي تغييرات في الرمز البرمجي (code).

الناقل يُرسل البريد الالكتروني بشكل غير متزامن بالفعل وفقاً لإعداد Messenger الافتراضي:

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 4
    :class: ignore

    framework:
        messenger:
            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async

                # Route your messages to the transports
                App\Message\CommentMessage: async

حتى إذا كنا نستخدم نفس النقل لرسائل التعليق ورسائل البريد الإلكتروني ، فلا يجب أن يكون الأمر كذلك. يمكنك أن تقرر استخدام وسيلة نقل أخرى لإدارة أولويات الرسائل المختلفة على سبيل المثال.يمنحك استخدام وسائل النقل المختلفة أيضًا فرصة وجود أجهزة عاملة مختلفة تتعامل مع أنواع مختلفة من الرسائل. إنه مرن و متروك لكم.

اختبار رسائل البريد الإلكتروني
---------------------------------------------------------

هناك طرق عديدة لاختبار رسائل البريد الإلكتروني.

يمكنك كتابة اختبارات الوحدة (unit tests) إذا كنت تكتب فئة (class) لكل بريد الكتروني (علي سبيل المثال بتمديد ``Email`` او ``TemplatedEmail``).

الاختبارات الأكثر شيوعًا التي ستكتبها هي الاختبارات الوظيفية التي تتحقق من أن بعض الإجراءات تؤدي إلى إرسال بريد إلكتروني ، وربما اختبارات حول محتوى رسائل البريد الإلكتروني إذا كانت ديناميكية.

يأتي سيمفوني مع تأكيدات (assertions) تُسهل من مثل هذه الاختبارات، هاك أحد الأمثلة لعرض بعض الاحتمالات:

.. code-block:: php
    :class: ignore

    public function testMailerAssertions(): void
    {
        $client = static::createClient();
        $client->request('GET', '/');

        $this->assertEmailCount(1);
        $event = $this->getMailerEvent(0);
        $this->assertEmailIsQueued($event);

        $email = $this->getMailerMessage(0);
        $this->assertEmailHeaderSame($email, 'To', 'fabien@example.com');
        $this->assertEmailTextBodyContains($email, 'Bar');
        $this->assertEmailAttachmentCount($email, 1);
    }

تعمل هذه التأكيدات عندما يتم إرسال رسائل البريد الإلكتروني بشكل متزامن أو غير متزامن.

إرسال رسائل بريد الكتروني علي سحابية سيمفوني (Upsun)
-------------------------------------------------------------------------------------------------

.. index::
    single: Upsun;Emails
    single: Upsun;Mailer
    single: Upsun;SMTP
    single: Emails

لا يوجد تكوين محدد لـ Upsun. تأتي جميع الحسابات مع حساب Sendgrid الذي يتم استخدامه تلقائيًا لإرسال رسائل البريد الإلكتروني.

.. index::
    single: Symfony CLI;cloud:env:info

.. note::

    لتكون علي الجانب الآمن، رسائل البريد الالكتروني يتم ارسالها فقط علي الفرع الرئيسي (``master`` branch) بشكل افتراضي (default). قم بتفعيل SMTP بشكل صريح علي الفروع غير الرئيسية (non-``master`` branches) إدا كنت تعرف ما تفعله.

    .. code-block:: terminal

        $ symfony cloud:env:info enable_smtp on

.. sidebar:: الذهاب أبعد من ذلك

    * `دروس باعث البريد الالكتروني علي SymfonyCasts`_؛

    * `مراجع لغة النماذج Inky`_؛

    * `مُعالجات متغير بيئة العمل`_؛

    * `مراجع باعث بريد إطار سيموفني`_؛

    * `مراجع Upsun عن رسائل البريد الإلكتروني`_.

.. _`Inky`: https://get.foundation/emails/docs/inky.html
.. _`دروس باعث البريد الالكتروني علي SymfonyCasts`: https://symfonycasts.com/screencast/mailer
.. _`مراجع لغة النماذج Inky`: https://get.foundation/emails/docs/inky.html
.. _`مُعالجات متغير بيئة العمل`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`مراجع باعث بريد إطار سيموفني`: https://symfony.com/doc/current/mailer.html
.. _`مراجع Upsun عن رسائل البريد الإلكتروني`: https://symfony.com/doc/current/cloud/services/emails.html
