Idziemy w asynchroniczność
============================

.. index::
    single: Async

Sprawdzenie, czy komentarz nie jest spamem, podczas obsługi wysyłania formularza może prowadzić do pewnych problemów. Jeśli model AI stanie się powolny, nasza strona również będzie powolna dla użytkowników. Co gorsza, jeżeli przekroczymy limit czasu lub model jest niedostępny, możemy stracić komentarze.

Idealnie byłoby, gdybyśmy przechowywali przesłane dane bez ich publikowania i natychmiast zwracali odpowiedź. Sprawdzenie, czy nie ma spamu, może zostać wykonane poza głównym wątkiem.

Oznaczanie komentarzy
---------------------

.. index::
    single: Command;make:entity

Musimy wprowadzić atrybut ``state`` określający stan komentarzy, przyjmujący trzy wartości: ``submitted`` dla komentarzy wysłanych, ``published`` dla opublikowanych oraz ``spam`` dla odrzuconych jako spamowe.

Dodaj atrybut ``state`` do klasy ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Powinniśmy także upewnić się, że domyślna wartość atrybutu ``state`` jest ustawiona na ``submitted``:

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

Utwórz migrację bazy danych:

.. code-block:: terminal

    $ symfony console make:migration

Zmodyfikuj migrację tak, by zaktualizowała status wszystkich istniejących komentarzy na ``published``:

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

Uruchom migrację bazy danych:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Zaktualizuj reguły wyświetlania, aby uniknąć pojawienia się nieopublikowanych komentarzy na stronie:

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

Zaktualizuj konfigurację EasyAdmin tak, by móc zobaczyć stan komentarza:

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

Nie zapomnij również zaktualizować fabryk testowych: komentarze tworzone przez ``CommentFactory`` powinny być domyślnie opublikowane, aby pojawiały się na stronach konferencji (test zawsze może nadpisać stan, gdy potrzebuje zmoderowanego komentarza):

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

W przypadku testów kontrolera należy przeprowadzić symulację walidacji:

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

Z poziomu testu PHPUnit można uzyskać dowolną usługę z kontenera za pośrednictwem ``self::getContainer()->get()``; daje to również dostęp do usług niepublicznych.

Zrozumienie komponentu Messenger
--------------------------------

.. index::
    single: Messenger
    single: Components;Messenger

Zarządzanie kodem asynchronicznym w Symfony jest zadaniem komponentu Messenger:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Gdy jakieś działania powinny być wykonywane asynchronicznie, wyślij *wiadomość* (ang. message) do *magistrali komunikacyjnej* (ang. messenger bus). Magistrala dodaje wiadomość do *kolejki* (ang. queue) i natychmiast zwraca wynik, aby umożliwić wykonywanie od razu kolejnych operacji.

*Konsument* (ang. consumer) pracuje stale w tle, czytając nowe wiadomości w kolejce i wykonuje związane z nimi schematy działań. Konsument może działać na tym samym serwerze co aplikacja internetowa lub osobnym.

Jest to bardzo podobne do sposobu obsługi żądań HTTP, z wyjątkiem tego, że nie mamy odpowiedzi.

Kodowanie obsługi wiadomości (ang. message handler)
-----------------------------------------------------

Wiadomość jest klasą obiektu danych, z którą nie są związane żadne reguły działania. Będzie ona zserializowana, aby być przechowywana w kolejce, więc przechowuj tylko "proste" dane, które można serializować.

Stwórz klasę ``CommentMessage``:

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

W świecie Messengera nie mamy kontrolerów, lecz obiekty obsługi wiadomości (ang. message handlers).

Utwórz klasę ``CommentMessageHandler`` w nowej przestrzeni nazw ``App\MessageHandler``, która wie, jak obsługiwać wiadomości ``CommentMessage``:

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

``AsMessageHandler`` pomaga Symfony tylko w automatycznej rejestracji i konfiguracji klasy do obsługi wiadomości. Zgodnie z konwencją, reguły obsługi znajdują się w metodzie ``__invoke()``. Podpowiedź typu ``CommentMessage`` do jednego z argumentów tej metody mówi Messengerowi, którą klasę będzie obsługiwać.

Zaktualizuj kontroler, aby korzystać z nowego systemu:

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

Zamiast polegać na kontrolerze spamu, wysyłamy teraz wiadomość do magistrali. Następnie obiekt obsługi decyduje, co z nią zrobić.

Osiągnęliśmy coś nieoczekiwanego. Odłączyliśmy nasz kontroler od kontrolera spamu i przenieśliśmy schemat działania do nowej klasy – obsługi (ang. handler). Jest to idealne zastosowanie dla magistrali. Przetestuj kod: działa. Wszystko jest nadal wykonywane synchronicznie, ale kod jest już prawdopodobnie "lepszy".

Idziemy w prawdziwą asynchroniczność
---------------------------------------

Domyślnie, obsługa wywoływana jest synchronicznie. Aby przejść do trybu asynchronicznego, należy skonfigurować, której kolejki użyć dla każdego obiektu obsługującego wiadomości (ang. handler) w pliku konfiguracyjnym ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

Konfiguracja każe magistrali wysyłać instancje ``App\Message\CommentMessage`` do kolejki ``async``, która jest zdefiniowana przez DSN (``MESSENGER_TRANSPORT_DSN``), przechowywany w pliku zmiennych środowiskowych ``.env``. Krótko mówiąc, używamy PostgreSQL jako kolejki dla naszych wiadomości.

.. tip::

    Za kulisami Symfony używa wbudowanego PostgreSQL, wydajnego, skalowalnego i transakcyjnego systemu pub/sub (``LISTEN``/``NOTIFY``). Możesz także przeczytać rozdział o RabbitMQ, jeśli chcesz go używać zamiast PostgreSQL jako brokera wiadomości.

Przetwarzanie wiadomości
-------------------------

Jeśli spróbujesz przesłać nowy komentarz, kontroler spamu nie będzie już wywoływany. Dodaj wywołanie ``error_log()`` w metodzie ``getSpamScore()``, aby to potwierdzić. Zamiast tego, wiadomość czeka w kolejce, gotowa do skonsumowania przez jakiś proces.

.. index::
    single: Command;messenger:consume

Jak możesz się domyślić, Symfony posiada polecenie przetwarzania wiadomości. Uruchom je teraz:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Powinno ono natychmiast przetworzyć wiadomość wysłaną w związku z przesłanymi komentarzami:

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

Aktywność konsumentów jest zapisywana w logach, ale możesz otrzymać natychmiastową informację zwrotną na konsoli, przekazując flagę``-vv``. Możesz dostrzec nawet połączenie do API OpenAI.

Aby zatrzymać konsumenta, naciśnij ``Ctrl+C``.

Robotnicy (ang. workers) działający w tle
-------------------------------------------

Zamiast uruchamiać konsumenta za każdym razem, gdy zamieszczamy komentarz, i zatrzymywać go natychmiast po tym, chcemy uruchomić go w sposób ciągły, bez otwierania zbyt wielu okien lub zakładek terminala.

Symfony CLI może zarządzać takimi poleceniami w tle (robotnikami) używając flagi demona (``-d``) na poleceniu ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Uruchom ponownie konsumenta wiadomości, ale umieść go w tle:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

Opcja ``--watch`` mówi Symfony, że polecenie musi zostać zrestartowane za każdym razem, gdy dochodzi do zmiany plików w katalogach ``config/``, ``src/``, ``templates/``, lub ``vendor/``.

.. note::

    Nie używaj ``-vv``, ponieważ otrzymasz zduplikowane wiadomości w ``server:log`` (wiadomości zarówno w logach i w konsoli).

Jeśli konsument przestanie pracować z jakiegoś powodu (limit pamięci, błąd, itp.), to zostanie automatycznie uruchomiony ponownie. Ale jeśli konsument przestanie pracować natychmiast po uruchomieniu – Symfony CLI podda się.

.. index::
    single: Symfony CLI;server:log

Logi są przesyłane strumieniowo przez ``symfony server:log`` wraz z wszystkimi innymi logami pochodzącymi z PHP, serwera WWW i aplikacji:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Użyj polecenia ``server:status``, aby wyświetlić listę wszystkich robotników pracujących w tle w bieżącym projekcie:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Aby zatrzymać robotnika, zatrzymaj serwer WWW lub zabij PID podany przez polecenie ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Ponawianie dostarczenia niedostarczonych wiadomości
----------------------------------------------------

A co, jeśli baza danych nie działa podczas przetwarzania wiadomości? Twórca komentarza nie ma o tym pojęcia, ale wiadomość kończy się niepowodzeniem i komentarz, którego dotyczyła, nie zostaje sprawdzony pod kątem bycia spamem.

Messenger posiada mechanizm ponawiania przetwarzania wiadomości w przypadku wystąpienia wyjątku podczas jej obsługi:

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

Jeśli pojawi się problem podczas obsługi wiadomości, konsument spróbuje ponownie trzy razy przed rezygnacją. A potem zamiast odrzucić wiadomość, umieści ją w trwałej kolejce nieudanych przetworzeń (ang. ``failed``), która wykorzystuje inną tabelę w bazie danych.

Przejrzyj nieudane wiadomości, a następnie ponów przetwarzanie za pomocą następujących poleceń:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Uruchamianie robotników na Upsun
---------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Aby przetwarzać wiadomości od PostgreSQL, musimy bezustannie uruchamiać polecenie ``messenger:consume``. Na Upsun jest to rola *robotnika* (ang. worker):

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Podobnie jak Symfony CLI, Upsun zarządza restartami i logami.

.. index::
    single: Symfony CLI;cloud:log

Aby odczytać logi robotnika, użyj:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Idąc dalej

    * `Samouczek SymfonyCasts Messenger`_;

    * 'Korporacyjna magistrala usług <https://pl.wikipedia.org/wiki/Enterprise_Service_Bus>'_ (ang. `enterprise service bus`_) oraz `wzorzec CQRS`_;

    * `Dokumentacja Symfony Messenger`_;

.. _`Samouczek SymfonyCasts Messenger`: https://symfonycasts.com/screencast/messenger
.. _`enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`wzorzec CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`Dokumentacja Symfony Messenger`: https://symfony.com/doc/current/messenger.html
