منع البريد العشوائي باستخدام الذكاء الاصطناعي
====================================================================================================

.. index::
    single: Spam

يمكن لأي شخص إرسال ملاحظات. حتى الروبوتات والبريد المزعج وأشياء أخرى. يمكننا إضافة "captcha" إلى النموذج حتى نحمي بطريقة ما من الروبوتات ، أو يمكننا استخدام بعض واجهات برمجة التطبيقات (API) لتباعيات خارجية.

لقد قررت استخدام نموذج لغوي كبير (Large Language Model) لتحديد ما إذا كان التعليق بريدًا عشوائيًا، لتوضيح كيفية استخدام الذكاء الاصطناعي في تطبيق Symfony وكيفية إجراء مثل هذه المكالمات المكلفة "خارج النطاق".

الحصول على مفتاح واجهة برمجة تطبيقات الذكاء الاصطناعي
----------------------------------------------------------------------------------

.. index::
    single: AI
    single: OpenAI

يدعم Symfony AI العديد من مزودي النماذج: OpenAI و Anthropic و Google Gemini و Mistral، وحتى النماذج المحلية عبر Ollama. يستخدم هذا الفصل OpenAI: قم بالتسجيل على `platform.openai.com`_ وأنشئ مفتاح API. إذا كنت تفضل مزودًا آخر، يبقى الكود كما هو؛ يتغير الإعداد فقط.

الاعتماد على Symfony AI Bundle
----------------------------------------------------------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

بدلاً من استدعاء واجهة HTTP الخاصة بالنموذج بأنفسنا، سنستخدم Symfony AI Bundle. يوفر تجريدًا لـ *منصة* (platform) لمزودي النماذج (يأتي كل مزود كحزمة جسر خاصة به) و *وكيل* (agent) يغلف نموذجًا لإجراء المكالمات؛ ويستفيد من جميع أدوات تصحيح Symfony مثل التكامل مع Symfony Profiler:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI مجموعة حديثة من المكونات وما زالت تجريبية: قد تتطور واجهاتها البرمجية أسرع من بقية Symfony.

لقد قامت وصفة جسر OpenAI (recipe) بإعداد المنصة لنا بالفعل؛ وهي تشير إلى متغير بيئة ``OPENAI_API_KEY`` (وأضافت قيمة افتراضية فارغة له في ``.env``):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

قم بإعداد *وكيل* (agent) افتراضي فوقها:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

استخدام متغيرات البيئة
------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

لا نريد بالتأكيد ترميز قيمة المفتاح بشكل ثابت في الإعداد؛ لهذا السبب تتم قراءته من متغير البيئة ``OPENAI_API_KEY``.

ومن ثم يعود الأمر لكل مطور لتعيين متغير بيئة "حقيقي" أو لتخزين القيمة في ملف ``env.local``:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

للإنتاج Production ، يجب تحديد متغير بيئة "حقيقي".

هذا يعمل بشكل جيد ، ولكن إدارة العديد من متغيرات البيئة قد تصبح مرهقة. في مثل هذه الحالة ، لدى Symfony بديل "أفضل" عندما يتعلق الأمر بتخزين الأسرار.

تخزين الأسرار
-------------------------

.. index::
    single: Secret

بدلاً من استخدام العديد من متغيرات البيئة ، يمكن لـ Symfony إدارة * vault * حيث يمكنك تخزين العديد من الأسرار. إحدى السمات الرئيسية هي القدرة على تنفيذ القبو vault  في المخزن Repository  (ولكن بدون المفتاح لفتحه). ميزة أخرى رائعة هي أنه يمكن إدارة قبو vault واحد لكل بيئة.

.. index:: ! Command;secrets:set

الأسرار هي متغيرات بيئية متخفية.

أضف مفتاح OpenAI API في الخزنة:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

نظرًا لأن هذه هي المرة الأولى التي نقوم فيها بتشغيل هذا الأمر ، فقد تم إنشاء مفتاحين في دليل ``/config/secret/dev``. ثم قام بتخزين سر ``OPENAI_API_KEY`` في نفس الدليل.

بالنسبة لأسرار التطوير ، يمكنك أن تقرر تنفيذ الخزنة والمفاتيح التي تم إنشاؤها في دليل ``/config/secret/dev``.

يمكن أيضًا تجاوز الأسرار عن طريق تعيين متغير بيئة يحمل نفس الاسم.

.. index::
    single: Command;secrets:reveal

لقراءة سر مرة أخرى من الخزنة، استخدم ``secrets:reveal``:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

تصميم فئة Spam Checker Class  للبريد المزعج
---------------------------------------------------------------

.. index::
    single: AI;Prompt

قم بإنشاء فئة جديدة تحت ``src/`` باسم ``SpamChecker`` لتغليف منطق سؤال النموذج عما إذا كان التعليق بريدًا عشوائيًا:

.. code-block:: php
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\AI\Agent\AgentInterface;
    use Symfony\AI\Platform\Exception\ExceptionInterface;
    use Symfony\AI\Platform\Message\Message;
    use Symfony\AI\Platform\Message\MessageBag;

    class SpamChecker
    {
        public function __construct(
            private AgentInterface $agent,
        ) {
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $messages = new MessageBag(
                Message::forSystem(<<<PROMPT
                    You moderate comments submitted to a conference guestbook.
                    Classify the comment as "ham", "maybe spam", or "blatant spam".
                    Only answer with the classification.
                    PROMPT),
                Message::ofUser(sprintf(<<<COMMENT
                    IP: %s
                    User agent: %s
                    Author: %s (%s)
                    Comment: %s
                    COMMENT,
                    $context['user_ip'] ?? '',
                    $context['user_agent'] ?? '',
                    $comment->getAuthor(),
                    $comment->getEmail(),
                    $comment->getText(),
                )),
            );

            try {
                $answer = strtolower($this->agent->call($messages)->getContent());
            } catch (ExceptionInterface) {
                // when the model cannot answer, let a human moderate the comment
                return 1;
            }

            return match (true) {
                str_contains($answer, 'blatant spam') => 2,
                str_contains($answer, 'maybe spam') => 1,
                default => 0,
            };
        }
    }

تخبر *موجِّه النظام* (system prompt) النموذجَ بدوره وتقيّد إجاباته؛ بينما تحتوي *رسالة المستخدم* (user message) على التعليق وسياق إرساله (عنوان IP، وكيل المستخدم).

تُظهر طريقة `` getSpamScore()  `` ثلاث  قيم بناءً على إجابة النموذج:

* ``2``: إذا كان التعليق "بريدًا عشوائيًا واضحًا" ؛

* ``1``: إذا كان التعليق قد يكون بريدًا عشوائيًا، أو عند تعذّر الوصول إلى النموذج ؛

* ``0``: إذا كان التعليق غير مرغوب فيه (ham).

إخراج النموذج نص حر، حتى عندما يقيّده الموجِّه: حلّله بتسامح (حوّله إلى أحرف صغيرة، استخدم ``str_contains()``). وعندما يتعذّر على النموذج الإجابة تمامًا، ارجع إلى الإشراف البشري بدلاً من الفشل: يجب أن يساعد الذكاء الاصطناعي المسؤول، لا أن يعطّل دفتر الزوار.

.. tip::

    حاول إرسال تعليق يبدو واضحًا أنه بريد عشوائي، مثل "Buy cheap watches at http://example.com/!!!"، لرؤية النموذج وهو يعمل.

التحقق من التعليقات على البريد المزعج
---------------------------------------------------------------------

تتمثل إحدى الطرق البسيطة للتحقق من البريد العشوائي عند إرسال تعليق جديد في الاتصال بمدقق البريد العشوائي قبل تخزين البيانات في قاعدة البيانات:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,7 +35,8 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -48,6 +50,17 @@ final class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +
    +            $context = [
    +                'user_ip' => $request->getClientIp(),
    +                'user_agent' => $request->headers->get('user-agent'),
    +                'referrer' => $request->headers->get('referer'),
    +                'permalink' => $request->getUri(),
    +            ];
    +            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    +                throw new \RuntimeException('Blatant spam, go away!');
    +            }
    +
                 $this->entityManager->flush();

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);

تحقق من أنه يعمل بشكل جيد.

تحديد معدل إرسال التعليقات
---------------------------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

يحمي اكتشاف البريد العشوائي الموقع من المرسلين المتطورين. هناك حماية تكميلية وأرخص بكثير وهي تقييد سرعة إرسال نفس العميل للتعليقات: لا أحد ينشر بشكل مشروع عشرات التعليقات في الساعة على دفتر زوار.

أضف مكون Symfony Rate Limiter:

.. code-block:: terminal

    $ symfony composer req rate-limiter

قم بإعداد مُحدِّد يقبل 5 تعليقات كحد أقصى في الساعة من نفس العميل:

.. code-block:: yaml
    :caption: config/packages/rate_limiter.yaml

    framework:
        rate_limiter:
            comment_submission:
                policy: 'fixed_window'
                limit: 5
                interval: '1 hour'

    when@test:
        framework:
            rate_limiter:
                comment_submission:
                    limit: 1000

تُرسل الاختبارات الآلية العديد من التعليقات بشكل مشروع في فترة زمنية قصيرة، لذلك يُرفع الحد لبيئة ``test``.

افرض المُحدِّد على إرسال التعليقات باستخدام السمة ``#[RateLimit]``؛ افتراضيًا، يُعرّف العملاء بعنوان IP الخاص بهم:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    +use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -31,6 +32,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,

لاحظ وسيط ``methods``: تصفح صفحة المؤتمر طلب ``GET`` ويجب ألا يُقيَّد؛ تُقيَّد فقط عمليات إرسال التعليقات (طلبات ``POST``).

عند بلوغ الحد، يُرجع Symfony تلقائيًا استجابة ``429 Too Many Requests`` مع ترويسة HTTP ``Retry-After`` تُخبر العميل متى يمكنه إعادة المحاولة.

يحمي المكون نفسه أيضًا نموذج تسجيل دخول المدير من هجمات القوة الغاشمة (brute-force)؛ تفعيل *خنق تسجيل الدخول* (login throttling) على جدار الحماية يحتاج سطرًا واحدًا:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -19,6 +19,7 @@ security:
             main:
                 lazy: true
                 provider: app_user_provider
    +            login_throttling: ~
                 form_login:
                     login_path: app_login
                     check_path: app_login

افتراضيًا، يحظر Symfony عنوان IP بعد 5 محاولات تسجيل دخول فاشلة لنفس اسم المستخدم خلال دقيقة (يعيد تسجيل الدخول الناجح ضبط العدّاد). استخدم خياري ``max_attempts`` و ``interval`` لضبط السياسة.

إدارة الأسرار في الإنتاج Production
--------------------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

للإنتاج ، يدعم Upsun إعداد *متغيرات البيئة الحساسة*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

ولكن كما نُقش أعلاه ، قد يكون استخدام أسرار Symfony أفضل. ليس من حيث الأمن ، ولكن من حيث الإدارة السرية لفريق المشروع. يتم تخزين جميع الأسرار في المخزن Repository  ومتغير البيئة الوحيد الذي تحتاج إلى إدارته للإنتاج هو مفتاح فك التشفير. وهذا يجعل من الممكن لأي شخص في الفريق إضافة أسرار الإنتاج حتى إذا لم يكن لديهم إمكانية الوصول إلى خوادم الإنتاج. التثبيت  أكثر صعوبة  نوعا ما بالرغم من ذلك.

.. index::
    single: Command;secrets:generate-keys

أولاً ، قم بإنشاء زوج من المفاتيح لاستخدام الإنتاج:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    على Linux وأنظمة التشغيل المشابهة، استخدم ``APP_RUNTIME_ENV=prod`` بدلاً من ``--env=prod`` لأن ذلك يتجنب تصريف التطبيق لبيئة ``prod``:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

أعد إضافة سر OpenAI API في قبو الإنتاج ولكن بقيمته للإنتاج:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

الخطوة الأخيرة هي إرسال مفتاح فك التشفير إلى Upsun عن طريق تعيين متغير حساس:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

يمكنك إضافة جميع الملفات وتنفيذها ؛ تمت إضافة مفتاح فك التشفير في .gitignore تلقائيًا ، لذلك لن يتم الالتزام به أبدًا. لمزيد من الأمان ، يمكنك إزالته من جهازك المحلي حيث تم نشره الآن:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: الذهاب أبعد من ذلك

    * `مستندات Symfony AI`_؛

    * `معالجات المتغيرة للبيئة`_؛

    * `كيفية الحفاظ على سرية المعلومات الحساسة`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`مستندات Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`معالجات المتغيرة للبيئة`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`كيفية الحفاظ على سرية المعلومات الحساسة`: https://symfony.com/doc/current/configuration/secrets.html
