آزمودن
==============

.. index::
    single: PHPUnit

همان‌طور که شروع به افزودن قابلیت‌های بیشتر و بیشتری به اپلیکیشن می‌کنیم، احتمالاً زمان مناسبی برای صحبت درباره‌ی آزمودن (testing) است.

*نکته‌ی جالب*: هنگام نوشتن آزمون‌های این فصل یک باگ پیدا کردم.

سیمفونی برای آزمون‌های واحد بر PHPUnit تکیه می‌کند. بیایید آن را نصب کنیم:

.. code-block:: terminal

    $ symfony composer req phpunit --dev

نوشتن آزمون‌های واحد
------------------------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` اولین کلاسی است که برای آن آزمون می‌نویسیم. یک آزمون واحد تولید کنید:

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

آزمودن SpamChecker یک چالش است زیرا قطعاً نمی‌خواهیم API مربوط به OpenAI را فراخوانی کنیم: این کار کند و پرهزینه خواهد بود و حتی پاسخ‌ها قطعی (deterministic) نخواهند بود. ما قصد داریم *platform* را با یک نمونه‌ی ساختگی جایگزین کنیم.

.. index::
    single: Mock

بیایید اولین آزمون را برای زمانی که مدل در دسترس نیست بنویسیم:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/SpamCheckerTest.php
    +++ w/tests/SpamCheckerTest.php
    @@ -2,12 +2,25 @@

     namespace App\Tests;

    +use App\Entity\Comment;
    +use App\SpamChecker;
     use PHPUnit\Framework\TestCase;
    +use Symfony\AI\Agent\Agent;
    +use Symfony\AI\Platform\Exception\RuntimeException;
    +use Symfony\AI\Platform\Test\InMemoryPlatform;

     class SpamCheckerTest extends TestCase
     {
    -    public function testSomething(): void
    +    public function testSpamScoreWhenTheModelIsDown(): void
         {
    -        $this->assertTrue(true);
    +        $comment = new Comment();
    +        $comment->setAuthor('Fabien');
    +        $comment->setEmail('fabien@example.com');
    +        $comment->setText('Such a nice conference!');
    +
    +        $platform = new InMemoryPlatform(fn () => throw new RuntimeException('The model is down.'));
    +        $checker = new SpamChecker(new Agent($platform, 'gpt-5-mini'));
    +
    +        $this->assertSame(1, $checker->getSpamScore($comment, []));
         }
     }

کلاس ``InMemoryPlatform`` رابط platform را بدون فراخوانی هیچ API بیرونی پیاده‌سازی می‌کند. با دادن یک callable، می‌تواند هر رفتاری، از جمله شکست‌ها را شبیه‌سازی کند. ما آن را در یک ``Agent`` واقعی در بر می‌گیریم تا منطق ``SpamChecker`` به صورت واقعی آزموده شود.

زمانی که مدل در دسترس نیست، کامنت‌ها باید به یک تعدیل‌کننده‌ی انسانی برسند: امتیاز موردانتظار ``1`` است.

آزمون‌ها را اجرا کنید تا بررسی کنید که قبول می‌شوند:

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

بیایید آزمون‌هایی را برای مسیر موفق (happy path) اضافه کنیم:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/SpamCheckerTest.php
    +++ w/tests/SpamCheckerTest.php
    @@ -4,6 +4,7 @@ namespace App\Tests;

     use App\Entity\Comment;
     use App\SpamChecker;
    +use PHPUnit\Framework\Attributes\DataProvider;
     use PHPUnit\Framework\TestCase;
     use Symfony\AI\Agent\Agent;
     use Symfony\AI\Platform\Exception\RuntimeException;
    @@ -23,4 +24,25 @@ class SpamCheckerTest extends TestCase

             $this->assertSame(1, $checker->getSpamScore($comment, []));
         }
    +
    +    #[DataProvider('provideComments')]
    +    public function testSpamScore(int $expectedScore, string $answer): void
    +    {
    +        $comment = new Comment();
    +        $comment->setAuthor('Fabien');
    +        $comment->setEmail('fabien@example.com');
    +        $comment->setText('Such a nice conference!');
    +
    +        $platform = new InMemoryPlatform($answer);
    +        $checker = new SpamChecker(new Agent($platform, 'gpt-5-mini'));
    +
    +        $this->assertSame($expectedScore, $checker->getSpamScore($comment, []));
    +    }
    +
    +    public static function provideComments(): iterable
    +    {
    +        yield 'blatant_spam' => [2, 'blatant spam'];
    +        yield 'maybe_spam' => [1, 'Maybe spam.'];
    +        yield 'ham' => [0, 'ham'];
    +    }
     }

تأمین‌کننده‌های داده (data providers) در PHPUnit به ما اجازه می‌دهند تا از منطق آزمون یکسانی برای چندین مورد آزمون استفاده مجدد کنیم.

نوشتن آزمون‌های کارکردی برای کنترلرها
--------------------------------------------------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

آزمودن کنترلرها کمی متفاوت از آزمودن یک کلاس PHP «معمولی» است، چرا که می‌خواهیم آن‌ها را در زمینه‌ی یک درخواست HTTP اجرا کنیم.

یک آزمون کارکردی برای کنترلر Conference ایجاد کنید:

.. code-block:: php
    :caption: tests/Controller/ConferenceControllerTest.php

    namespace App\Tests\Controller;

    use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

    class ConferenceControllerTest extends WebTestCase
    {
        public function testIndex(): void
        {
            $client = static::createClient();
            $client->request('GET', '/');

            $this->assertResponseIsSuccessful();
            $this->assertSelectorTextContains('h2', 'Give your feedback');
        }
    }

استفاده از ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` به جای ``PHPUnit\Framework\TestCase`` به عنوان کلاس پایه برای آزمون‌هایمان، یک انتزاع خوب برای آزمون‌های کارکردی به ما می‌دهد.

متغیر ``$client`` یک مرورگر را شبیه‌سازی می‌کند. اما به جای انجام فراخوانی‌های HTTP به سرور، مستقیماً اپلیکیشن سیمفونی را فراخوانی می‌کند. این راهبرد چند مزیت دارد: بسیار سریع‌تر از داشتن رفت‌وبرگشت بین کلاینت و سرور است، اما همچنین به آزمون‌ها اجازه می‌دهد تا پس از هر درخواست HTTP، وضعیت سرویس‌ها را بررسی کنند.

این اولین آزمون بررسی می‌کند که صفحه‌ی اصلی یک پاسخ HTTP با کد ۲۰۰ بازمی‌گرداند.

ادعاهایی (assertions) مانند ``assertResponseIsSuccessful`` علاوه بر PHPUnit برای تسهیل کار شما اضافه شده‌اند. سیمفونی ادعاهای بسیار زیادی از این دست تعریف کرده است.

.. tip::

    ما برای URL از ``/`` به جای تولید آن از طریق مسیریاب استفاده کردیم. این کار عمداً انجام شده است زیرا آزمودن URLهای کاربر نهایی بخشی از چیزی است که می‌خواهیم بیازماییم. اگر مسیر راه را تغییر دهید، آزمون‌ها شکست می‌خورند، که یادآوری خوبی است که احتمالاً باید URL قدیمی را به URL جدید بازهدایت کنید تا با موتورهای جستجو و وب‌سایت‌هایی که به وب‌سایت شما پیوند می‌دهند، مهربان باشید.

پیکربندی محیط آزمون
------------------------------------------------

.. index::
    single: Symfony Environments

به صورت پیش‌فرض، آزمون‌های PHPUnit در محیط سیمفونیِ ``test`` اجرا می‌شوند، همان‌طور که در فایل پیکربندی PHPUnit تعریف شده است:

.. code-block:: xml
    :caption: phpunit.xml.dist
    :emphasize-lines: 4
    :class: ignore

    <phpunit>
        <php>
            <ini name="error_reporting" value="-1" />
            <server name="APP_ENV" value="test" force="true" />
            <server name="SHELL_VERBOSITY" value="-1" />
            <server name="SYMFONY_PHPUNIT_REMOVE" value="" />
            <server name="SYMFONY_PHPUNIT_VERSION" value="8.5" />
            ...

.. index:: Command;secrets:set

برای اینکه آزمون‌ها کار کنند، باید رمز ``OPENAI_API_KEY`` را برای این محیط ``test`` تنظیم کنیم:

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

کار با یک پایگاه‌داده‌ی آزمون
------------------------------------------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

همان‌طور که پیش‌تر دیدیم، رابط خط فرمان سیمفونی به صورت خودکار متغیر محیط ``DATABASE_URL`` را در اختیار می‌گذارد. زمانی که ``APP_ENV`` برابر ``test`` است، مانند زمانی که PHPUnit اجرا می‌شود، نام پایگاه‌داده از ``app`` به ``app_test`` تغییر می‌کند تا آزمون‌ها پایگاه‌داده‌ی مخصوص خودشان را داشته باشند:

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

این موضوع بسیار مهم است زیرا برای اجرای آزمون‌هایمان به برخی داده‌های پایدار نیاز خواهیم داشت و قطعاً نمی‌خواهیم آنچه را که در پایگاه‌داده‌ی توسعه ذخیره کرده‌ایم بازنویسی کنیم.

پیش از آنکه بتوانیم آزمون را اجرا کنیم، باید پایگاه‌داده‌ی ``test`` را «مقداردهی اولیه» کنیم (پایگاه‌داده را ایجاد و migrate کنیم):

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    بر روی Linux و سیستم‌عامل‌های مشابه، می‌توانید به جای ``--env=test`` از
    ``APP_ENV=test`` استفاده کنید:

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

اگر اکنون آزمون‌ها را اجرا کنید، PHPUnit دیگر با پایگاه‌داده‌ی توسعه‌ی شما تعامل نخواهد داشت. برای اجرای تنها آزمون‌های جدید، مسیر کلاس آن‌ها را پاس دهید:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    زمانی که یک آزمون شکست می‌خورد، بررسی شیء Response می‌تواند مفید باشد. به آن از طریق ``$client->getResponse()`` دسترسی پیدا کنید و آن را ``echo`` کنید تا ببینید چه شکلی است.

تعریف Factoryها
----------------------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

برای اینکه بتوانیم فهرست کامنت‌ها، صفحه‌بندی و ارسال فرم را بیازماییم، نیاز داریم که پایگاه‌داده را با مقداری داده پر کنیم. و برای آنکه آزمون‌ها مستقل از یکدیگر بمانند، هر آزمون باید دقیقاً مجموعه‌داده‌ای را که نیاز دارد ایجاد کند. *factoryهای شیء* ابزار مناسبی برای این کار هستند.

Zenstruck Foundry را نصب کنید:

.. code-block:: terminal

    $ symfony composer req foundry --dev

برای هر موجودیتی که آزمون‌ها به آن نیاز دارند یک factory تولید کنید:

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

یک factory توصیف می‌کند که چگونه یک موجودیت معتبر بسازد: به لطف کتابخانه‌ی Faker برای هر ویژگی یک مقدار پیش‌فرض تولید می‌شود. ایجاد یک شیء از طریق یک factory آن را persist نیز می‌کند. مقادیر پیش‌فرض کنفرانس را واقع‌گرایانه‌تر تنظیم کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Factory/ConferenceFactory.php
    +++ w/src/Factory/ConferenceFactory.php
    @@ -34,9 +34,9 @@ final class ConferenceFactory extends PersistentObjectFactory
     protected function defaults(): array|callable
     {
         return [
    -        'city' => self::faker()->text(255),
    +        'city' => self::faker()->city(),
             'isInternational' => self::faker()->boolean(),
    -        'slug' => self::faker()->text(255),
    -        'year' => self::faker()->text(4),
    +        'slug' => '-',
    +        'year' => self::faker()->year(),
         ];
     }

تنظیم slug روی ``-`` به شنونده‌ی موجودیتی که هنگام افزودن slugها نوشتیم اجازه می‌دهد تا مقدار واقعی را محاسبه کند: کنفرانسی که با شهر ``Amsterdam`` و سال ``2019`` ساخته شود، به‌صورت خودکار slug ``amsterdam-2019`` را می‌گیرد.

همین کار را برای کامنت‌ها انجام دهید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Factory/CommentFactory.php
    +++ w/src/Factory/CommentFactory.php
    @@ -34,10 +34,10 @@ final class CommentFactory extends PersistentObjectFactory
     protected function defaults(): array|callable
     {
         return [
    -        'author' => self::faker()->text(255),
    +        'author' => self::faker()->name(),
             'conference' => ConferenceFactory::new(),
             'createdAt' => \DateTimeImmutable::createFromMutable(self::faker()->dateTime()),
    -        'email' => self::faker()->text(255),
    +        'email' => self::faker()->email(),
             'text' => self::faker()->text(),
         ];
     }

به مقدار پیش‌فرض ``conference`` توجه کنید: زمانی که یک کامنت بدون کنفرانس صریح ساخته شود، Foundry یکی را در لحظه ایجاد می‌کند.

پیمایش یک وب‌سایت در آزمون‌های کارکردی
------------------------------------------------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

همان‌طور که دیدیم، کلاینت HTTP استفاده‌شده در آزمون‌ها یک مرورگر را شبیه‌سازی می‌کند، بنابراین می‌توانیم در وب‌سایت مانند استفاده از یک مرورگر بدون‌سر (headless) پیمایش کنیم.

یک آزمون جدید اضافه کنید که از صفحه‌ی اصلی روی یک صفحه‌ی کنفرانس کلیک می‌کند:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -2,10 +2,17 @@

     namespace App\Tests\Controller;

    +use App\Factory\CommentFactory;
    +use App\Factory\ConferenceFactory;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    +use Zenstruck\Foundry\Test\Factories;
    +use Zenstruck\Foundry\Test\ResetDatabase;

     class ConferenceControllerTest extends WebTestCase
     {
    +    use Factories;
    +    use ResetDatabase;
    +
         public function testIndex(): void
         {
             $client = static::createClient();
    @@ -14,4 +21,24 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseIsSuccessful();
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }
    +
    +    public function testConferencePage(): void
    +    {
    +        $client = static::createClient();
    +
    +        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    +        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $amsterdam]);
    +
    +        $crawler = $client->request('GET', '/');
    +
    +        $this->assertCount(2, $crawler->filter('h4'));
    +
    +        $client->clickLink('View');
    +
    +        $this->assertPageTitleContains('Amsterdam');
    +        $this->assertResponseIsSuccessful();
    +        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    +        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +    }
     }

trait مربوط به ``Factories``، factoryها را در آزمون‌ها فعال می‌کند و ``ResetDatabase`` پایگاه‌داده را در آغاز هر اجرای آزمون بازنشانی می‌کند.

بیایید آنچه را که در این آزمون رخ می‌دهد به زبان ساده توصیف کنیم:

* آزمون دقیقاً مجموعه‌داده‌ای را که نیاز دارد ایجاد می‌کند: دو کنفرانس و یک کامنت، از طریق factoryها؛

* مانند اولین آزمون، به صفحه‌ی اصلی می‌رویم؛

* متد ``request()`` یک نمونه‌ی ``Crawler`` بازمی‌گرداند که در یافتن عناصر روی صفحه (مانند پیوندها، فرم‌ها یا هر چیزی که با CSS selector یا XPath قابل دسترسی است) کمک می‌کند؛

* به لطف یک CSS selector، ادعا می‌کنیم که دو کنفرانس روی صفحه‌ی اصلی فهرست شده‌اند؛

* سپس روی پیوند «View» کلیک می‌کنیم (از آنجایی که نمی‌تواند بیش از یک پیوند را در یک زمان کلیک کند، سیمفونی به صورت خودکار اولین پیوندی را که پیدا می‌کند انتخاب می‌کند)؛

* عنوان صفحه، پاسخ و ``<h2>`` صفحه را ادعا می‌کنیم تا مطمئن شویم در صفحه‌ی درست هستیم (همچنین می‌توانستیم راهِ منطبق را بررسی کنیم)؛

* در نهایت، ادعا می‌کنیم که ۱ کامنت روی صفحه وجود دارد. ``div:contains()`` یک CSS selector معتبر نیست، اما سیمفونی برخی افزوده‌های خوب دارد که از jQuery قرض گرفته شده‌اند.

به جای کلیک‌کردن روی متن (یعنی ``View``)، می‌توانستیم پیوند را از طریق یک CSS selector نیز انتخاب کنیم:

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

بررسی کنید که آزمون جدید سبز است:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

ارسال یک فرم در یک آزمون کارکردی
------------------------------------------------------------

می‌خواهید به سطح بعدی بروید؟ سعی کنید با شبیه‌سازی ارسال یک فرم، یک کامنت جدید به همراه یک عکس روی یک کنفرانس از درون یک آزمون اضافه کنید. بلندپروازانه به نظر می‌رسد، اینطور نیست؟ به کد موردنیاز نگاه کنید: پیچیده‌تر از آنچه پیش‌تر نوشتیم نیست:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -41,4 +41,23 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
             $this->assertSelectorExists('div:contains("There are 1 comments")');
         }
    +
    +    public function testCommentSubmission(): void
    +    {
    +        $client = static::createClient();
    +
    +        $berlin = ConferenceFactory::createOne(['city' => 'Berlin', 'year' => '2021', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $berlin]);
    +
    +        $client->request('GET', '/conference/berlin-2021');
    +        $client->submitForm('Submit', [
    +            'comment[author]' => 'Fabien',
    +            'comment[text]' => 'Some feedback from an automated functional test',
    +            'comment[email]' => 'me@automat.ed',
    +            'comment[photo]' => dirname(__DIR__, 2).'/public/images/under-construction.gif',
    +        ]);
    +        $this->assertResponseRedirects();
    +        $client->followRedirect();
    +        $this->assertSelectorExists('div:contains("There are 2 comments")');
    +    }
     }

برای ارسال یک فرم از طریق ``submitForm()``، نام ورودی‌ها را به لطف DevTools مرورگر یا از طریق پنل فرم در نمایه‌ساز سیمفونی پیدا کنید. به استفاده‌ی هوشمندانه‌ی مجدد از تصویر «در دست احداث» توجه کنید!

آزمون‌ها را دوباره اجرا کنید تا بررسی کنید که همه‌چیز سبز است:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

اگر می‌خواهید نتیجه را در یک مرورگر بررسی کنید، وب سرور را متوقف کرده و آن را برای محیط ``test`` دوباره اجرا کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

اجرای دوباره‌ی آزمون‌ها
----------------------------------------

اگر آزمون‌ها را برای بار دوم اجرا کنید، همچنان قبول می‌شوند: trait مربوط به ``ResetDatabase`` پایگاه‌داده را در آغاز هر اجرای آزمون بازنشانی می‌کند و هر آزمون دقیقاً مجموعه‌داده‌ای را که نیاز دارد ایجاد می‌کند. هیچ وضعیت مشترکی و هیچ باقی‌مانده‌ای از اجرای قبلی وجود ندارد:

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

خودکارسازی گردش‌کار با یک Makefile
------------------------------------------------------------

.. index::
    single: Makefile

به‌خاطرسپردن یک دنباله از فرمان‌ها برای اجرای آزمون‌ها آزاردهنده است. این موضوع باید حداقل مستند شود. اما مستندسازی باید آخرین راه‌حل باشد. به جای آن، نظرتان درباره‌ی خودکارسازی فعالیت‌های روزمره چیست؟ این کار هم به عنوان مستندات عمل می‌کند، هم به کشف توسط سایر توسعه‌دهندگان کمک می‌کند و هم زندگی توسعه‌دهندگان را آسان‌تر و سریع‌تر می‌کند.

استفاده از یک ``Makefile`` یک راه برای خودکارسازی فرمان‌هاست:

.. code-block:: makefile
    :caption: Makefile

    SHELL := /bin/bash

    tests:
    	symfony console doctrine:database:drop --force --env=test || true
    	symfony console doctrine:database:create --env=test
    	symfony console doctrine:migrations:migrate -n --env=test
    	symfony php bin/phpunit $(MAKECMDGOALS)
    .PHONY: tests

.. warning::

    در یک قاعده‌ی Makefile، تورفتگی **باید** از یک کاراکتر tab منفرد به جای فاصله‌ها تشکیل شده باشد.

به پرچم ``-n`` در فرمان Doctrine توجه کنید؛ این یک پرچم سراسری در فرمان‌های سیمفونی است که آن‌ها را غیرتعاملی می‌کند.

هرگاه خواستید آزمون‌ها را اجرا کنید، از ``make tests`` استفاده کنید:

.. code-block:: terminal

    $ make tests

بازنشانی پایگاه‌داده پس از هر آزمون
--------------------------------------------------------------------

.. index::
    single: PHPUnit;Performance

بازنشانی پایگاه‌داده پس از هر اجرای آزمون خوب است، اما داشتن آزمون‌های واقعاً مستقل حتی بهتر است. ما نمی‌خواهیم یک آزمون به نتایج آزمون‌های قبلی تکیه کند. تغییردادن ترتیب آزمون‌ها نباید نتیجه را تغییر دهد. همان‌طور که اکنون متوجه خواهیم شد، در حال حاضر این‌گونه نیست.

آزمون ``testConferencePage`` را به بعد از ``testCommentSubmission`` منتقل کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -22,26 +22,6 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertSelectorTextContains('h2', 'Give your feedback');
         }

    -    public function testConferencePage(): void
    -    {
    -        $client = static::createClient();
    -
    -        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    -        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    -        CommentFactory::createOne(['conference' => $amsterdam]);
    -
    -        $crawler = $client->request('GET', '/');
    -
    -        $this->assertCount(2, $crawler->filter('h4'));
    -
    -        $client->clickLink('View');
    -
    -        $this->assertPageTitleContains('Amsterdam');
    -        $this->assertResponseIsSuccessful();
    -        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    -        $this->assertSelectorExists('div:contains("There are 1 comments")');
    -    }
    -
         public function testCommentSubmission(): void
         {
             $client = static::createClient();
    @@ -41,5 +22,25 @@ class ConferenceControllerTest extends WebTestCase
             $this->assertResponseRedirects();
             $client->followRedirect();
             $this->assertSelectorExists('div:contains("There are 2 comments")');
         }
    +
    +    public function testConferencePage(): void
    +    {
    +        $client = static::createClient();
    +
    +        $amsterdam = ConferenceFactory::createOne(['city' => 'Amsterdam', 'year' => '2019', 'isInternational' => true]);
    +        ConferenceFactory::createOne(['city' => 'Paris', 'year' => '2020', 'isInternational' => false]);
    +        CommentFactory::createOne(['conference' => $amsterdam]);
    +
    +        $crawler = $client->request('GET', '/');
    +
    +        $this->assertCount(2, $crawler->filter('h4'));
    +
    +        $client->clickLink('View');
    +
    +        $this->assertPageTitleContains('Amsterdam');
    +        $this->assertResponseIsSuccessful();
    +        $this->assertSelectorTextContains('h2', 'Amsterdam 2019');
    +        $this->assertSelectorExists('div:contains("There are 1 comments")');
    +    }
     }

اکنون آزمون‌ها شکست می‌خورند.

.. index::
    single: Doctrine;TestBundle

برای بازنشانی پایگاه‌داده بین آزمون‌ها، DoctrineTestBundle را نصب کنید:

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

شما باید اجرای recipe را تأیید کنید (زیرا یک باندل «رسماً» پشتیبانی‌شده نیست):

.. code-block:: text
    :class: ignore

    Symfony operations: 1 recipe (a5c79a9ff21bc3ae26d9bb25f1262ed7)
      -  WARNING  dama/doctrine-test-bundle (>=4.0): From github.com/symfony/recipes-contrib:master
        The recipe for this package comes from the "contrib" repository, which is open to community contributions.
        Review the recipe at https://github.com/symfony/recipes-contrib/tree/master/dama/doctrine-test-bundle/4.0

        Do you want to execute this recipe?
        [y] Yes
        [n] No
        [a] Yes for all packages, only for the current installation session
        [p] Yes permanently, never ask again for this project
        (defaults to n): p

و تمام. هر تغییری که در آزمون‌ها انجام شود، اکنون به صورت خودکار در پایان هر آزمون بازگردانده (roll-back) می‌شود.

آزمون‌ها باید دوباره سبز شوند:

.. code-block:: terminal

    $ make tests

استفاده از یک مرورگر واقعی برای آزمون‌های کارکردی
-----------------------------------------------------------------------------------

.. index::
    single: Test;Panther
    single: Panther

آزمون‌های کارکردی از یک مرورگر ویژه استفاده می‌کنند که مستقیماً لایه‌ی سیمفونی را فراخوانی می‌کند. اما به لطف Symfony Panther می‌توانید از یک مرورگر واقعی و لایه‌ی HTTP واقعی نیز استفاده کنید:

.. code-block:: terminal

    $ symfony composer req panther --dev

سپس می‌توانید آزمون‌هایی بنویسید که با تغییرات زیر از یک مرورگر واقعی Google Chrome استفاده می‌کنند:

.. code-block:: diff
    :class: ignore

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -2,13 +2,13 @@

     namespace App\Tests\Controller;

    -use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
    +use Symfony\Component\Panther\PantherTestCase;

    -class ConferenceControllerTest extends WebTestCase
    +class ConferenceControllerTest extends PantherTestCase
     {
         public function testIndex(): void
         {
    -        $client = static::createClient();
    +        $client = static::createPantherClient(['external_base_uri' => rtrim($_SERVER['SYMFONY_PROJECT_DEFAULT_ROUTE_URL'], '/')]);
             $client->request('GET', '/');

             $this->assertResponseIsSuccessful();

متغیر محیط ``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` حاوی URL وب سرور محلی است.

انتخاب نوع درست آزمون
------------------------------------------------

.. index::
    single: Command;make:test

ما تا اینجا سه نوع متفاوت از آزمون ایجاد کرده‌ایم. با اینکه تنها از باندل maker برای تولید کلاس آزمون واحد استفاده کردیم، می‌توانستیم از آن برای تولید سایر کلاس‌های آزمون نیز استفاده کنیم:

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

باندل maker بسته به اینکه می‌خواهید اپلیکیشن خود را چگونه بیازمایید، از تولید انواع آزمون‌های زیر پشتیبانی می‌کند:

* ``TestCase``: آزمون‌های پایه‌ی PHPUnit؛

* ``KernelTestCase``: آزمون‌های پایه‌ای که به سرویس‌های سیمفونی دسترسی دارند؛

* ``WebTestCase``: برای اجرای سناریوهای مرورگرگونه، اما بدون اجرای کد JavaScript؛

* ``ApiTestCase``: برای اجرای سناریوهای API-محور؛

* ``PantherTestCase``: برای اجرای سناریوهای e2e، با استفاده از یک مرورگر واقعی یا کلاینت HTTP و یک وب سرور واقعی.

اجرای آزمون‌های کارکردی جعبه‌سیاه با Blackfire
------------------------------------------------------------------------------------

یک راه دیگر برای اجرای آزمون‌های کارکردی، استفاده از `Blackfire player`_ است. علاوه بر آنچه می‌توانید با آزمون‌های کارکردی انجام دهید، می‌تواند آزمون‌های کارایی را نیز انجام دهد.

برای کسب اطلاعات بیشتر، گام :doc:`کارایی <29-performance>` را بخوانید.

.. sidebar:: بیشتر بدانید

    * `فهرست ادعاهای تعریف‌شده توسط سیمفونی`_ برای آزمون‌های کارکردی؛

    * `مستندات PHPUnit`_؛

    * `مستندات Foundry`_؛

    * کتابخانه‌ی `Faker`_ برای تولید داده‌های fixture واقع‌گرایانه؛

    * `مستندات کامپوننت CssSelector`_؛

    * کتابخانه‌ی `Symfony Panther`_ برای آزمودن مرورگر و پیمایش وب در اپلیکیشن‌های سیمفونی؛

    * `مستندات Make/Makefile`_.

.. _`Blackfire player`: https://blackfire.io/player
.. _`فهرست ادعاهای تعریف‌شده توسط سیمفونی`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`مستندات PHPUnit`: https://phpunit.de/documentation.html
.. _`مستندات Foundry`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`Faker`: https://github.com/FakerPHP/Faker
.. _`مستندات کامپوننت CssSelector`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`مستندات Make/Makefile`: https://www.gnu.org/software/make/manual/make.html
