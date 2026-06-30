测试
====

.. index::
    single: PHPUnit

随着我们开始往应用程序里添加越来越多的功能，现在大概是谈论测试的好时机了。

*趣闻*：我在写本章的测试时发现了一个 bug。

Symfony 依赖 PHPUnit 来做单元测试。我们来安装它：

.. code-block:: terminal

    $ symfony composer req phpunit --dev

编写单元测试
------------------

.. index::
    single: Test;Unit Tests
    single: Unit Tests
    single: Command;make:test

``SpamChecker`` 是我们要为之编写测试的第一个类。生成一个单元测试：

.. code-block:: terminal

    $ symfony console make:test TestCase SpamCheckerTest

测试 SpamChecker 是个有挑战的工作，因为我们当然不想去真的调用 OpenAI 的 API：那样会很慢、很贵，而且回答甚至都不是确定的。我们要用一个假的 *platform* 来替换它。

.. index::
    single: Mock

我们先来为模型无法连接的情况写一个测试：

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

``InMemoryPlatform`` 类实现了 platform 接口，但不会调用任何外部 API。给定一个可调用对象（callable），它可以模拟任何行为，包括失败。我们把它包装在一个真实的 ``Agent`` 里，这样 ``SpamChecker`` 的逻辑就能被真正地测试。

当模型下线时，评论必须交给人来审核：期望的分值是 ``1``。

运行测试来检查它们是否通过：

.. code-block:: terminal

    $ symfony php bin/phpunit

.. index::
    single: PHPUnit;Data Provider
    single: Data Provider
    single: Attributes;DataProvider

我们来为正常情况（happy path）添加测试：

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

PHPUnit 的数据提供器（data provider）让我们可以对多个测试用例重用同一套测试逻辑。

为控制器编写功能测试
------------------------------

.. index::
    single: Test;Functional Tests
    single: Functional Tests
    single: Components;Browser Kit
    single: Browser Kit

测试控制器和测试一个“普通的” PHP 类有点不同，因为我们想要在 HTTP 请求的上下文里运行它们。

为会议控制器创建一个功能测试：

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

用 ``Symfony\Bundle\FrameworkBundle\Test\WebTestCase`` 而不是 ``PHPUnit\Framework\TestCase`` 来作为我们测试的基类，这为功能测试提供了一个不错的抽象。

``$client`` 变量模拟了一个浏览器。但它不是向服务器发起 HTTP 调用，而是直接调用 Symfony 应用程序。这个策略有几个好处：它比客户端和服务器之间的往返要快得多，但它也允许测试在每个 HTTP 请求后检视各个服务的状态。

第一个测试检查首页是否返回一个 200 的 HTTP 应答。

诸如 ``assertResponseIsSuccessful`` 这样的断言是在 PHPUnit 之上添加的，用来方便你的工作。Symfony 定义了很多这样的断言。

.. tip::

    我们用了 ``/`` 作为 URL，而不是通过路由器来生成它。这是有意为之的，因为测试最终用户的 URL 也是我们想要测试的一部分。如果你改变了路由路径，测试就会失败，这是一个很好的提醒：你大概应该把旧 URL 重定向到新 URL，以便对搜索引擎以及链接到你网站的其它网站友好。

配置测试环境
------------------

.. index::
    single: Symfony Environments

默认情况下，PHPUnit 测试运行在 ``test`` 这个 Symfony 环境里，这是在 PHPUnit 配置文件中定义的：

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

为了让测试能工作，我们必须为这个 ``test`` 环境设置 ``OPENAI_API_KEY`` 机密信息：

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY --env=test

使用一个测试数据库
------------------------

.. index::
    single: Test;Database
    single: Functional Tests,Database

正如我们已经看到的，``symfony`` 命令会自动暴露 ``DATABASE_URL`` 环境变量。当 ``APP_ENV`` 为 ``test`` 时（运行 PHPUnit 时就是这样设置的），数据库名会从 ``app`` 变成 ``app_test``，这样测试就有它们自己的数据库：

.. code-block:: yaml
    :class: ignore
    :emphasize-lines: 5
    :caption: config/packages/doctrine.yaml

    when@test:
        doctrine:
            dbal:
                # "TEST_TOKEN" is typically set by ParaTest
                dbname_suffix: '_test%env(default::TEST_TOKEN)%'

这非常重要，因为我们需要一些稳定的数据来运行测试，而且我们当然不想覆盖存储在开发数据库里的内容。

在能够运行测试之前，我们需要“初始化” ``test`` 数据库（创建数据库并迁移它）：

.. code-block:: terminal

    $ symfony console doctrine:database:create --env=test
    $ symfony console doctrine:migrations:migrate -n --env=test

.. note::

    在 Linux 和类似的操作系统上，你可以用 ``APP_ENV=test`` 来代替
    ``--env=test``：

    .. code-block:: terminal
        :class: ignore

        $ APP_ENV=test symfony console doctrine:database:create

如果你现在运行测试，PHPUnit 就不会再和你的开发数据库交互了。如果只想运行新的测试，传入它们的类路径：

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

.. tip::

    当一个测试失败时，检视 Response 对象可能会有用。通过 ``$client->getResponse()`` 访问它，并把它 ``echo`` 出来，看看它是什么样子。

定义工厂
------------

.. index::
    single: Foundry
    single: Fixtures
    single: Test;Factories
    single: Command;make:factory

为了能够测试评论列表、分页和表单提交，我们需要往数据库里填充一些数据。而为了让各个测试彼此独立，每个测试都应该创建它所需要的那一套确切的数据。*对象工厂（object factory）* 是完成这项工作的完美工具。

安装 Zenstruck Foundry：

.. code-block:: terminal

    $ symfony composer req foundry --dev

为测试所需的每个实体生成一个工厂：

.. code-block:: terminal

    $ symfony console make:factory Conference

.. code-block:: terminal

    $ symfony console make:factory Comment

一个工厂描述了如何构建一个合法的实体：借助 Faker 库，每个属性都会生成一个默认值。通过工厂创建一个对象也会把它持久化。把会议的默认值调得更真实一些：

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

把 slug 设置为 ``-`` 会让我们在添加 slug 时写的实体监听器计算出真正的值：一个用城市 ``Amsterdam`` 和年份 ``2019`` 创建的会议会自动得到 ``amsterdam-2019`` 这个 slug。

对评论也做同样的处理：

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

注意 ``conference`` 这个默认值：当一条评论在没有显式指定会议的情况下被创建时，Foundry 会即时创建一个会议。

在功能测试中爬取网站
------------------------------

.. index::
    single: Components;CssSelector
    single: Components;DomCrawler
    single: Test;Crawling
    single: Crawling

正如我们所见，测试里用的 HTTP 客户端模拟了一个浏览器，所以我们可以像使用一个无头浏览器那样在网站里导航。

添加一个新测试，它从首页点击进入一个会议页面：

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

``Factories`` trait 在测试里启用了工厂，``ResetDatabase`` 在每次测试运行开始时重置数据库。

我们用大白话来描述下这个测试里发生了什么：

* 测试通过工厂创建了它所需的那套确切数据：两个会议和一条评论；

* 和第一个测试一样，我们访问首页；

* ``request()`` 方法返回一个 ``Crawler`` 实例，它帮助在页面上查找元素（比如链接、表单，或者任何你能用 CSS 选择器或 XPath 找到的东西）；

* 借助一个 CSS 选择器，我们断言首页上列出了两个会议；

* 然后我们点击 “View” 链接（由于一次不能点击多于一个链接，Symfony 会自动选择它找到的第一个）；

* 我们断言页面标题、应答以及页面的 ``<h2>``，来确保我们在正确的页面上（我们也可以检查匹配的路由）；

* 最后，我们断言页面上有 1 条评论。``div:contains()`` 不是一个合法的 CSS 选择器，但 Symfony 有一些不错的扩展，是从 jQuery 借鉴来的。

我们也可以不点击文字（即 ``View``），而是通过一个 CSS 选择器来选取这个链接：

.. code-block:: php
    :class: ignore

    $client->click($crawler->filter('h4 + p a')->link());

检查这个新测试是否通过：

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

在功能测试中提交表单
------------------------------

你想要更进一步吗？试着在一个测试里通过模拟表单提交，给一个会议添加一条带照片的新评论。这看上去很有野心，不是吗？看看所需的代码：并不比我们已经写过的更复杂：

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

要通过 ``submitForm()`` 提交一个表单，借助浏览器的开发者工具，或者通过 Symfony 分析器的 Form 面板来找到 input 的名字。注意这里巧妙地重用了“在建设中”的图片！

再次运行测试来检查是否一切都通过：

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

如果你想在浏览器里查看结果，停止 web 服务器，然后为 ``test`` 环境重新运行它：

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ APP_ENV=test symfony server:start -d

.. figure:: screenshots/tests-add-comment.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

再次运行测试
------------------

如果你第二次运行测试，它们仍然通过：``ResetDatabase`` trait 在每次测试运行开始时重置数据库，而每个测试都创建它所需的那套确切数据。没有共享状态，也没有上次运行遗留的内容：

.. code-block:: terminal

    $ symfony php bin/phpunit tests/Controller/ConferenceControllerTest.php

用 Makefile 来自动化你的工作流
----------------------------------------

.. index::
    single: Makefile

必须记住一连串命令才能运行测试是很烦人的。这至少应该被记录下来。但文档应该是最后的手段。相反，把日常活动自动化怎么样？那既可以作为文档，又能帮助其他开发者发现它，还能让开发者的生活更轻松更快捷。

使用 ``Makefile`` 是自动化命令的一种方式：

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

    在 Makefile 的规则里，缩进 **必须** 由单个制表符（tab）组成，而不是空格。

注意 Doctrine 命令上的 ``-n`` 选项；它是 Symfony 命令上的一个全局选项，让命令变成非交互式的。

每当你想运行测试时，使用 ``make tests``：

.. code-block:: terminal

    $ make tests

在每个测试后重置数据库
------------------------------

.. index::
    single: PHPUnit;Performance

在每次测试运行后重置数据库很不错，但让测试真正彼此独立就更好了。我们不想让一个测试依赖于之前测试的结果。改变测试的顺序不应该改变结果。正如我们现在将要发现的，目前情况还不是这样。

把 ``testConferencePage`` 测试移动到 ``testCommentSubmission`` 之后：

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

测试现在失败了。

.. index::
    single: Doctrine;TestBundle

为了在测试之间重置数据库，安装 DoctrineTestBundle：

.. code-block:: terminal
    :class: hide

    $ symfony composer config extra.symfony.allow-contrib true

.. code-block:: terminal

    $ symfony composer req "dama/doctrine-test-bundle:^8" --dev

你需要确认执行这个 recipe（因为它不是一个“官方”支持的 bundle）：

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

好了。现在测试里所做的任何改动都会在每个测试结束时自动回滚。

测试应该又变绿了：

.. code-block:: terminal

    $ make tests

用一个真实的浏览器来做功能测试
---------------------------------------

.. index::
    single: Test;Panther
    single: Panther

功能测试使用一个特殊的浏览器，它直接调用 Symfony 层。但借助 Symfony Panther，你也可以使用一个真实的浏览器和真实的 HTTP 层：

.. code-block:: terminal

    $ symfony composer req panther --dev

接着你就可以通过下面的改动来编写使用真实 Google Chrome 浏览器的测试：

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

``SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` 环境变量包含了本地 web 服务器的 URL。

选择正确的测试类型
------------------------

.. index::
    single: Command;make:test

到目前为止我们已经创建了三种不同类型的测试。虽然我们只用 maker bundle 生成了单元测试类，但我们也可以用它来生成其它测试类：

.. code-block:: terminal
    :class: ignore

    $ symfony console make:test WebTestCase Controller\\ConferenceController

    $ symfony console make:test PantherTestCase Controller\\ConferenceController

根据你想要怎样测试你的应用，maker bundle 支持生成以下几种类型的测试：

* ``TestCase``：基本的 PHPUnit 测试；

* ``KernelTestCase``：能访问 Symfony 服务的基本测试；

* ``WebTestCase``：运行类似浏览器的场景，但不执行 JavaScript 代码；

* ``ApiTestCase``：运行面向 API 的场景；

* ``PantherTestCase``：运行端到端（e2e）场景，使用真实浏览器或 HTTP 客户端以及真实的 web 服务器。

用 Blackfire 运行黑盒功能测试
------------------------------------------

运行功能测试的另一种方式是使用 `Blackfire 播放器`_。除了你用功能测试能做的事情之外，它还能执行性能测试。

阅读 :doc:`性能 <29-performance>` 那一步来了解更多。

.. sidebar:: 深入学习

    * 用于功能测试的 `Symfony 定义的断言列表`_；

    * `PHPUnit 文档`_；

    * `Foundry 文档`_；

    * 用来生成真实伪数据的 `Faker 库`_；

    * `CssSelector 组件文档`_；

    * 在 Symfony 应用中用于浏览器测试和网页爬取的 `Symfony Panther`_ 库；

    * `Make/Makefile 文档`_。

.. _`Blackfire 播放器`: https://blackfire.io/player
.. _`Symfony 定义的断言列表`: https://symfony.com/doc/current/testing/functional_tests_assertions.html
.. _`PHPUnit 文档`: https://phpunit.de/documentation.html
.. _`Foundry 文档`: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
.. _`Faker 库`: https://github.com/FakerPHP/Faker
.. _`CssSelector 组件文档`: https://symfony.com/doc/current/components/css_selector.html
.. _`Symfony Panther`: https://github.com/symfony/panther
.. _`Make/Makefile 文档`: https://www.gnu.org/software/make/manual/make.html
