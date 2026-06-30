جلوگیری از ارسال محتوای هرز (Spam) با کمک هوش مصنوعی
==============================================================================

.. index::
    single: Spam

هر کسی می‌تواند یک بازخورد ارسال کند. حتی ربات‌ها، تولیدکنندگان محتواهای هرز و غیره. می‌توانیم یک «کپچا (captcha)» به فرم بیافزاییم تا در برابر ربات‌ها محافظت شویم یا اینکه از APIهای شخص ثالث استفاده کنیم.

تصمیم گرفته‌ام از یک مدل زبانی بزرگ (Large Language Model) برای تصمیم‌گیری درباره‌ی هرز بودن یک کامنت استفاده کنم تا نشان دهم که چگونه می‌توان از هوش مصنوعی در یک اپلیکیشن سیمفونی استفاده کرد و چگونه می‌توان چنین فراخوانی‌های پرهزینه‌ای را به «خارج از باند (out of band)» منتقل کرد.

دریافت یک کلید API برای هوش مصنوعی
------------------------------------------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI از فراهم‌کنندگان مدل بسیاری پشتیبانی می‌کند: OpenAI، Anthropic، Google Gemini، Mistral و حتی مدل‌های محلی از طریق Ollama. این فصل از OpenAI استفاده می‌کند: در `platform.openai.com`_ ثبت‌نام کرده و یک کلید API بسازید. اگر فراهم‌کننده‌ی دیگری را ترجیح می‌دهید، کد یکسان می‌ماند؛ تنها پیکربندی تغییر می‌کند.

تکیه بر باندل Symfony AI
--------------------------------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

به جای فراخوانی API مربوط به HTTP مدل توسط خودمان، از باندل Symfony AI استفاده می‌کنیم. این باندل یک انتزاع *platform* برای فراهم‌کنندگان مدل ارائه می‌دهد (هر فراهم‌کننده به‌صورت بسته‌ی پل (bridge) مخصوص خودش می‌آید) و یک *agent* که یک مدل را برای انجام فراخوانی‌ها در بر می‌گیرد؛ و از تمام ابزارهای اشکال‌زدایی سیمفونی همچون یکپارچگی با نمایه‌ساز سیمفونی بهره می‌برد:

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI مجموعه‌ای جوان از کامپوننت‌ها و همچنان آزمایشی است: APIهای آن ممکن است سریع‌تر از باقی سیمفونی تغییر کنند.

recipe مربوط به پل OpenAI، platform را از پیش برای ما پیکربندی کرده است؛ این پیکربندی به یک متغیر محیط ``OPENAI_API_KEY`` ارجاع می‌دهد (و یک مقدار پیش‌فرض خالی برای آن در ``.env`` اضافه کرده است):

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

یک *agent* پیش‌فرض روی آن پیکربندی کنید:

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

استفاده از متغیر‌های محیط
------------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

ما مطمئناً نمی‌خواهیم که مقدار کلید را در پیکربندی هاردکد کنیم؛ به همین دلیل آن از متغیر محیط ``OPENAI_API_KEY`` خوانده می‌شود.

پس از این هر توسعه‌دهنده وظیفه دارد تا یک متغیر محیط «واقعی» را تنظیم کند یا مقدار آن را در فایل ``.env.local`` ذخیره کند:

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

برای محیط عمل‌آوری، باید یک متغیر محیط «واقعی» تعریف شود.

این روش به خوبی کار می‌کند، اما مدیریت تعداد زیادی متغیر محیط ممکن است مایه‌ی زحمت شود. در این صورت سیمفونی برای ذخیره‌ی رمز‌ها، یک راه جایگزین «بهتر» دارد.

ذخیره‌ی رمز‌ها (Secrets)
---------------------------------------

.. index::
    single: Secret

به جای استفاده از تعداد زیادی متغیر محیط، سیمفونی می‌تواند یک *گاوصندوق (vault)* که می‌توانید تعداد زیادی رمز را در آن ذخیره کنید، مدیریت کند. یک ویژگی کلیدی این روش قابلیت commitکردن گاوصندوق در مخزن Git است (اما بدون کلید که آن را باز می‌کند). یک ویژگی عالی دیگر این است که سیمفونی می‌تواند به ازای هر محیط یک گاوصندوق را مدیریت کند.

.. index:: ! Command;secrets:set

رمز‌ها همان متغیرهای محیط در لباس مبدل هستند.

کلید OpenAI API را به گاوصندوق اضافه کنید:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

چون این اولین باری است که این فرمان را اجرا می‌کنیم، فرمان دو عدد کلید را در پوشه‌ی ``config/secret/dev/`` تولید می‌کند. سپس رمز ``OPENAI_API_KEY`` را نیز در همان پوشه ذخیره می‌کند.

برای رمز‌های محیط توسعه، می‌توانید تصمیم بگیرید که گاوصندوق را به همراه کلیدهایی که برایش تولید شده و در پوشه‌ی ``config/secret/dev/`` قرار دارد، commit کنید.

همچنین رمزها می‌توانند از طریق تنظیم یک متغیر محیط با نام یکسان، بازنویسی (override) شوند.

.. index::
    single: Command;secrets:reveal

برای خواندن یک رمز از گاوصندوق، از ``secrets:reveal`` استفاده کنید:

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

طراحی یک کلاس بررسی‌کننده‌ی محتوای هرز
-------------------------------------------------------------------------

.. index::
    single: AI;Prompt

به منظور جای دادن منطق پرسش از مدل درباره‌ی هرز بودن یک کامنت، یک کلاس جدید در پوشه‌ی ``src/`` و با نام ``SpamChecker`` ایجاد نمایید:

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

*system prompt* به مدل نقش آن را می‌گوید و پاسخ‌هایش را محدود می‌کند؛ *user message* حاوی کامنت و زمینه‌ی ارسال آن (آدرس IP، user agent) است.

متد ``getSpamScore()`` با توجه به پاسخ مدل، می‌تواند ۳ مقدار مختلف برگرداند:

* ``2``: اگر کامنت آشکارا یک محتوای هرز باشد؛

* ``1``: اگر کامنت امکان هرز بودن داشته باشد، یا زمانی که مدل در دسترس نباشد؛

* ``0``: اگر کامنت هرز نباشد (ham).

خروجی یک مدل، متن آزاد است، حتی زمانی که prompt آن را محدود می‌کند: آن را با انعطاف تجزیه کنید (با حروف کوچک، با استفاده از ``str_contains()``). و زمانی که مدل اصلاً نمی‌تواند پاسخ دهد، به جای شکست‌خوردن به تعدیل انسانی پناه ببرید: هوش مصنوعی باید به مدیر کمک کند، نه اینکه هرگز guestbook را مسدود کند.

.. tip::

    سعی کنید کامنتی که آشکارا هرز به نظر می‌رسد ارسال کنید، مانند «Buy cheap watches at http://example.com/!!!»، تا مدل را در حال کار ببینید.

بررسی کامنت‌ها برای یافتن محتوای هرز
--------------------------------------------------------------------

هنگامی که یک کامنت جدید ارسال می‌شود، یک راه آسان برای بررسی هرز بودن محتوا، فراخوانی بررسی‌کننده‌ی محتوای هرز (spam checker) قبل از ذخیره‌ی کامنت در پایگاه‌داده است:

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

بررسی کنید که این روش به درستی کار می‌کند.

محدودسازی نرخ ارسال کامنت‌ها
-----------------------------------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

تشخیص محتوای هرز، وب‌سایت را در برابر اسپمرهای پیچیده محافظت می‌کند. یک محافظت مکمل و بسیار کم‌هزینه‌تر، محدودکردن سرعتی است که یک کلاینت می‌تواند کامنت ارسال کند: هیچ‌کس به‌طور قانونی ده‌ها کامنت در ساعت روی یک guestbook ارسال نمی‌کند.

کامپوننت Symfony Rate Limiter را اضافه کنید:

.. code-block:: terminal

    $ symfony composer req rate-limiter

یک محدودکننده پیکربندی کنید که حداکثر ۵ کامنت در ساعت از یک کلاینت بپذیرد:

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

آزمون‌های خودکار به‌طور قانونی تعداد زیادی کامنت را در بازه‌ی زمانی کوتاهی ارسال می‌کنند، بنابراین این محدودیت برای محیط ``test`` افزایش داده شده است.

محدودکننده را روی ارسال کامنت‌ها با attribute‌ی ``#[RateLimit]`` اعمال کنید؛ به‌صورت پیش‌فرض، کلاینت‌ها را با آدرس IP آن‌ها شناسایی می‌کند:

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

به آرگمان ``methods`` توجه کنید: مرور یک صفحه‌ی کنفرانس یک درخواست ``GET`` است و نباید محدود شود؛ تنها ارسال کامنت‌ها (درخواست‌های ``POST``) محدود می‌شوند.

زمانی که به محدودیت رسیده شود، سیمفونی به‌صورت خودکار یک پاسخ ``429 Too Many Requests`` به همراه سربرگ HTTP ``Retry-After`` بازمی‌گرداند که به کلاینت می‌گوید چه زمانی می‌تواند دوباره تلاش کند.

همین کامپوننت همچنین فرم ورود مدیر را در برابر حملات brute-force محافظت می‌کند؛ فعال‌سازی *throttling ورود* روی دیوارآتش تنها یک خط می‌خواهد:

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

به‌صورت پیش‌فرض، سیمفونی پس از ۵ تلاش ناموفق ورود برای یک نام کاربری در یک دقیقه، یک IP را مسدود می‌کند (یک ورود موفق شمارنده را بازنشانی می‌کند). از گزینه‌های ``max_attempts`` و ``interval`` برای تنظیم این سیاست استفاده کنید.

مدیریت رمز‌ها در محیط عمل‌آوری
----------------------------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

در محیط عمل‌آوری، Upsun از تنظیم *متغیرهای محیط حساس* پشتیبانی می‌کند:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

اما همانطور که در بالا بحث شد، استفاده از رمزهای سیمفونی می‌تواند راه بهتری باشد. البته نه از لحاظ امنیت، بلکه از نظر سهولت مدیریت رمز برای تیم پروژه. به این ترتیب، تمام رمزها در مخزن ذخیره شده و تنها متغیر محیط که نیاز دارید در محیط عمل‌آوری مدیریت کنید، کلید رمزگشایی است. این روش این امکان را به وجود می‌آورد که هر یک از اعضای تیم بدون آنکه به سرورهای عمل‌آوری دسترسی داشته باشند، بتوانند رمزهای عمل‌آوری اضافه کنند. البته راه‌اندازی آن کمی پیچیده‌تر است.

.. index::
    single: Command;secrets:generate-keys

ابتدا یک جفت کلید برای استفاده در محیط عمل‌آوری تولید کنید:

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    بر روی Linux و سیستم‌عامل‌های مشابه، به جای ``--env=prod`` از ``APP_RUNTIME_ENV=prod`` استفاده کنید زیرا این کار از کامپایل اپلیکیشن برای محیط ``prod`` جلوگیری می‌کند:

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

مجدداً رمز کلید OpenAI API را به گاوصندوق عمل‌آوری اضافه کنید اما با مقدار آن در محیط عمل‌آوری:

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

آخرین گام، ارسال کلید رمزگشایی به Upsun از طریق تنظیم یک متغیر محیط حساس می‌باشد:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

می‌توانید تمام فایل‌ها را اضافه کرده و commit کنید؛ کلید رمزگشایی به صورت خودکار به فایل ``.gitignore`` اضافه شده است، و بنابراین هرگز commit نخواهد شد. برای ایمنی بیشتر، می‌توانید کلید را از روی رایانه‌ی محلی خود پاک نمایید زیرا که دیگر مستقر شده است:

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: بیشتر بدانید

    * `مستندات Symfony AI`_؛

    * `پردازشگر متغیرهای محیط`_؛

    * `چگونه اطلاعات حساس را محرمانه نگه داریم`_.

.. _`platform.openai.com`: https://platform.openai.com
.. _`مستندات Symfony AI`: https://symfony.com/doc/current/ai/index.html
.. _`پردازشگر متغیرهای محیط`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`چگونه اطلاعات حساس را محرمانه نگه داریم`: https://symfony.com/doc/current/configuration/secrets.html
