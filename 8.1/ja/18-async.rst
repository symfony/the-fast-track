非同期にする
==================

.. index::
    single: Async

フォーム投稿時にスパムの判定をするのには多少問題があります。例えば、 AI モデルに遅延の問題があったときに、私たちの Web サイトも遅くなってしまいます。さらに、タイムアウトされてしまったり、モデルに問題があったときには、コメントを失ってしまうかもしれません。

公開することなく投稿されたデータを保存して、レスポンスを早く返すことが理想とするところです。そのためにスパムのチェックとは独立して実行します。

コメントにフラグを付ける
------------------------------------

.. index::
    single: Command;make:entity

コメントに ``submitted``, ``spam``, ``published`` という ``state`` を追加する必要があります。

``Comment`` クラスに ``state`` プロパティを追加しましょう:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

また、 ``state`` のデフォルトの値を ``submitted`` としてセットされていることを確認してください:

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

データベースマイグレーションを追加する:

.. code-block:: terminal

    $ symfony console make:migration

既に登録されているコメント全てに、デフォルトの値として ``published`` を指定するようにマイグレーションを修正してください:

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

データベースをマイグレートする:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

表示ロジックを変更して、公開されていないコメントをフロントエンドへ表示しないようにしましょう:

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

EasyAdmin の設定を変更してコメントの状態(state)を見ることができるようにしましょう。

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

テストのファクトリーも修正しましょう。 ``CommentFactory`` で作成されるコメントは、カンファレンスページに表示されるように、デフォルトで公開済みにしておきます（モデレート中のコメントが必要なテストでは、いつでも state を上書きできます）:

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

コントローラーのテストでは、バリデーションをシミュレートします:

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

PHPUnit のテストからは ``self::getContainer()->get()`` を使えば、全てのサービス（非公開なサービスも含めて）を取得することができます。

メッセンジャーを理解する
------------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Symfony で非同期処理を管理するために、メッセンジャーコンポーネントを使用します:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

非同期処理が必要な際に、 *メッセージ* を *メッセージバス* に送ってください。メッセンジャーバスはメッセージを *キュー* に保存したらすぐにリターンして、処理を待たせることなく速やかに再開させます。

*コンシューマー* は、継続的ににバックグラウンドで動いており、キューにある新しいメッセージを読み、そのメッセージに関連したロジックを実行します。コンシューマーは、Web アプリケーションと同じサーバーでも別のサーバーにあっても動作します。

レスポンスがない以外は、HTTP リクエストを処理するときととても似ています。

メッセージハンドラーをコーディングする
---------------------------------------------------------

メッセージはデータオブジェクトのクラスで、ロジックを持つべきではありません。シリアライズされ、キューに格納されます。 "シンプルな"シリアライズ可能なデータのみを格納しましょう。

``CommentMessage`` クラスを作成する:

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

メッセンジャーを使う際には、コントローラーではなくメッセージハンドラーが処理を担います。

``App\MessageHandler`` ネームスペース以下に ``CommentMessage`` メッセージを処理する ``CommentMessageHandler`` クラスを作成してください:

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

``AsMessageHandler `` は、そのクラスをメッセンジャーのハンドラとして Symfony が自動登録し設定できるようにします。規約では、ハンドラーのロジックは、 ``__invoke()`` メソッドに書きます。このメソッドの引数の ``CommentMessage`` 型宣言から、どのクラスを処理するのか知ることができます。

コントローラーを修正します:

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

スパムチェッカーに依存するのではなく、メッセージバスにディスパッチするようになりました。そして、ハンドラーにどう処理するかを決めさせます。

コントローラーとスパムチェックを隔離し、ロジックを新しいクラスのハンドラーに移動しました。バスの良いユースケースです。コードをテストして、動作するか確認してください。全て同期的に実行されますが、コードは、 "ベター"になっています。

実際に非同期にする
---------------------------

デフォルトでは、ハンドラは、同期的に処理します。非同期にするために、``config/packages/messenger.yaml`` の設定ファイルに、ハンドラがどのキューを使用するかを明示的に設定してください:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

この設定では、バスに ``App\Message\CommentMessage`` のインスタンスを ``非同期`` にキューへ送るようにしています。 非同期キューは環境変数 ``MESSENGER_TRANSPORT_DSN`` によって定義されており、 ``.env`` でDoctrineを指すように設定されています。つまり、メッセージのキューとして、PostgreSQLを利用しています。

.. tip::

    舞台裏では、SymfonyはPostgreSQLに組み込みの、高速で、スケーラブルで、トランザクションできる pub/sub システム(``LISTEN``/``NOTIFY``)を利用しています。メッセージの保存先としてPostgreSQLの代わりにRabbitMQを使いたい場合は、RabbitMQの章を読んでみてください。

メッセージを取得実行する
------------------------------------

新しくコメントを投稿しても、スパムチェッカーは呼ばれなくなりました。 ``getSpamScore()`` メソッドで ``error_log()`` 関数を追加して確認してみてください。代わりにキューにメッセージが入るようになったので、他のプロセスから取得され実行される準備ができました。

.. index::
    single: Command;messenger:consume

もうお分かりかもしれませんが、Symfony は、メッセージを取得し、実行するコマンドがビルトインされていますので、実行してみましょう:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

コマンドを実行するとすぐに、コメント送信でディスパッチされたメッセージが取得され、実行されるはずです:

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

メッセージ取得実行の処理がログに書かれますが、 ``--vv`` フラグを渡すことでコンソールに即時的なフィードバックを得ることができます。さらに、OpenAI API の呼び出しを探すこともできます。

メッセージの取得実行は、``Ctrl+C`` でストップします。

ワーカーをバックグラウンドで実行する
------------------------------------------------------

コメントを投稿した際に、毎回メッセージ取得の起動と停止を行うのではなく、ターミナルのウィンドウやタブを開くことなく、継続的に実行するようにしましょう。

Symfony CLI は、 ``run`` コマンドに ``-d`` フラグを付けることでデーモンとすることができ、こういったバックグラウンドで実行するコマンドを管理することができます。

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

メッセージ取得実行をもう一度走らせてください。今度はバックグラウンドで送信しましょう:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

``--watch`` オプションを付けることで、``config/``, ``src/``, ``templates/``, ``vendor/`` ディレクトリ内のファイルシステムに変更があった際に、Symfony にコマンドをリスタートさせることができます。

.. note::

    ``server:log`` でメッセージを重複させたくない際は、``--vv`` オプションは使用しないでください（ログされたメッセージとコンソールのメッセージ）。

メモリ制限やバグなどでメッセージの取得実行が停止した際は、自動的に再起動します。また、メッセージの取得実行の失敗が暴走した際は、 Symfony CLI は処理を停止します。

.. index::
    single: Symfony CLI;server:log

``symfony server:log`` コマンドで、PHP やWebサーバー、アプリケーションの全てのログのストリームを見ることができます:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

``server:status`` コマンドを使えば、現在のプロジェクトで管理されているバックグランドのワーカーの全ての一覧を表示できます:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

ワーカーを停止するには、Webサーバーを止めるか、``server:status`` コマンドで得られる PID をキルしてください:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

メッセージの失敗をリトライする
---------------------------------------------

メッセージ取得実行の際に、データベースが落ちていたらどうしますか？コメントの投稿者には何も影響はありませんが、メッセージは失敗し、スパムはチェックされません。

メッセンジャーには、メッセージのハンドリングで例外になったらリトライする機構があります:

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

メッセージのハンドリングに問題が起きた際に、メッセージの取得実行は諦めるまでに3回リトライをします。ただし、メッセージを廃棄するのではなく、恒久的に ``failed`` キューに保存します。failedキューは通常のキューとは別のデータベーステーブルを利用します。

失敗したメッセージを調べ、再実行するには次のコマンドを使用します:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Upsun でワーカーを実行する
------------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

PostgreSQL からメッセージ取得実行をするには、 ``messenger:consume`` コマンドを継続的に実行する必要があります。これは Upsun の *ワーカー* の役割です:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Symfony CLI のように、Upsun マネージャーはログをリスタートします。

.. index::
    single: Symfony CLI;cloud:log

ワーカーのログを取得するには、以下のようにしてください:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: より深く学ぶために

    * `SymfonyCasts Messenger チュートリアル`_;

    * `Enterprise サービスバス`_ アーキテクチャと `CQRS パターン`_;

    * `Symfony Messenger ドキュメント`_;

.. _`SymfonyCasts Messenger チュートリアル`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise サービスバス`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS パターン`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony Messenger ドキュメント`: https://symfony.com/doc/current/messenger.html
