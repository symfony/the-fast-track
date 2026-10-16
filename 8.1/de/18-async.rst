Asynchrone Verarbeitung
=======================

.. index::
    single: Async

Die Überprüfung auf Spam während der Bearbeitung des übermittelten Formulars kann zu Problemen führen. Wenn das KI-Modell langsam ist, wird unsere Website auch für Benutzer*innen langsam. Aber noch schlimmer: Wir könnten Kommentare verlieren, falls wir in einen Timeout laufen oder das Modell nicht verfügbar ist.

Im Idealfall sollten wir die übermittelten Daten speichern, ohne sie zu veröffentlichen, und sofort eine Response zurückliefern. Die Überprüfung auf Spam kann dann unabhängig davon durchgeführt werden.

Kommentare kennzeichnen
-----------------------

.. index::
    single: Command;make:entity

Wir müssen ein ``state``-Feld für Kommentare einführen: ``submitted``, ``spam`` und ``published``.

Füge die ``state``-Property zur ``Comment``-Klasse hinzu:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Wir sollten auch sicherstellen, dass der ``state``-Wert standardmäßig auf ``submitted`` gesetzt ist:

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

Erstelle eine Datenbankmigration:

.. code-block:: terminal

    $ symfony console make:migration

Passe die Migration an, um alle vorhandenen Kommentare standardmäßig auf ``published`` zu setzen:

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

Führe die Datenbankmigration durch:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Aktualisiere die Anzeigelogik, um zu vermeiden, dass unveröffentlichte Kommentare im Frontend erscheinen:

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

Aktualisiere die EasyAdmin-Konfiguration, um den Zustand des Kommentars zu sehen:

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

Denk daran, auch die Test-Factories zu aktualisieren: Kommentare, die von der ``CommentFactory`` erstellt werden, sollten standardmäßig veröffentlicht sein, damit sie auf den Konferenzseiten erscheinen (ein Test kann den Status jederzeit überschreiben, wenn er einen moderierten Kommentar braucht):

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

Simuliere die Validierung für die Controller-Tests:

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

Du kannst von einem PHPUnit-Test aus jeden beliebigen Service aus den Container über ``self::getContainer()->get()`` holen; dies ermöglicht auch den Zugriff auf Services, die nicht public sind.

Messenger verstehen
-------------------

.. index::
    single: Messenger
    single: Components;Messenger

Die asynchrone Verarbeitung mit Symfony ist Aufgabe der Messenger-Komponente:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Wenn Logik asynchron ausgeführt werden soll, sende eine *Message* (Nachricht) an einen *Messenger-Bus*. Der Bus speichert die Message in einer *Queue* (Warteschlange) und kehrt sofort zurück, um den Betriebsablauf so schnell wie möglich wieder aufzunehmen.

Ein *Consumer* läuft kontinuierlich im Hintergrund, um neue Messages auf der Queue zu lesen und die zugehörige Logik auszuführen. Der Consumer kann auf dem gleichen Server wie die Webanwendung oder auf einem separaten Server laufen.

Das Ganze ist der Art und Weise, wie HTTP-Requests behandelt werden sehr ähnlich, nur dass wir keine Responses zurückliefern.

Einen Message Handler erstellen
-------------------------------

Eine Message ist eine Datenobjektklasse, die keine Logik enthalten sollte. Sie wird serialisiert, um in einer Queue gespeichert zu werden, also speichere darin nur "einfache" serialisierbare Daten.

Lege die ``CommentMessage``-Klasse an:

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

In der Messenger-Welt haben wir keine Controller, sondern Message-Handler.

Erstelle eine ``CommentMessageHandler``-Klasse unter einem neuen ``App\MessageHandler``-Namespace, die weiß, wie man mit ``CommentMessage``-Messages umgeht:

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

``AsMessageHandler`` hilft Symfony, die Klasse automatisch zu registrieren und automatisch als Messenger-Handler zu konfigurieren. Nach Konvention lebt die Logik eines Handlers in einer Methode namens ``__invoke()``. Der ``CommentMessage``-Type-Hint auf das eine Argument dieser Methode sagt dem Messenger, welche Klasse diese verarbeiten soll.

Aktualisiere den Controller, damit er das neue System verwendet:

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

Anstatt vom Spam Checker abhängig zu sein, senden wir nun eine Message zum Bus. Der Handler entscheidet dann, was er damit macht.

Wir haben etwas Unerwartetes erreicht. Wir haben unseren Controller vom Spam Checker entkoppelt und die Logik in eine neue Klasse, den Handler, verschoben. Es ist ein perfekter Anwendungsfall für den Bus. Teste den Code, er funktioniert. Alles wird noch synchron gemacht, aber der Code ist wahrscheinlich schon "besser".

Echt Asynchron
--------------

Standardmäßig werden Handler synchron aufgerufen. Um asynchron zu werden, musst Du in der ``config/packages/messenger.yaml``-Konfigurationsdatei für jeden Handler explizit konfigurieren, welche Queue verwendet werden soll:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

Die Konfiguration weist den Bus an, Instanzen von ``App\Message\CommentMessage`` an die ``async``-Queue zu senden, die durch einen DSN (``MESSENGER_TRANSPORT_DSN``) definiert ist, welche wiederum zu Doctrine verweist, wie in ``.env`` definiert. Auf gut Deutsch: wir nutzen PostgreSQL als Warteschlange (queues) für unsere Messages (Nachrichten).

.. tip::

    Hinter den Kulissen nutzt Symfony das in PostgreSQL integrierte, performante, skalierbare und transaktionale pub/sub system (``LISTEN``/``NOTIFY``). Du kannst auch das RabbitMQ-Kapitel lesen, wenn Du das lieber als Message-Händler benutzt als PostgreSQL.

Messages verarbeiten
--------------------

Wenn Du versuchst, einen neuen Kommentar abzugeben, wird der Spam-Checker nicht mehr aufgerufen. Füge einen ``error_log()``-Aufruf in der ``getSpamScore()``-Methode hinzu, um Dich zu vergewissern. Stattdessen wartet in der Warteschlange (queue) eine Message (Nachricht), die von bestimmten Prozessen verarbeitet werden kann.

.. index::
    single: Command;messenger:consume

Selbstverständlich wird Symfony mit einem Verarbeitungsbefehl (Consumer Command) geliefert. Führe diesen jetzt aus:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Er sollte die für den eingereichten Kommentar versendete Message sofort verarbeiten:

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

Die Aktivität des Message Consumers wird geloggt, aber Du erhältst sofortiges Feedback auf der Konsole, indem Du das ``-vv`` Flag übergibst. Du solltest sogar den Aufruf der OpenAI-API sehen können.

Drücke ``Ctrl+C``, um den Consumer zu stoppen.

Worker im Hintergrund ausführen
--------------------------------

Anstatt den Consumer jedes Mal zu starten, wenn wir einen Kommentar posten und ihn sofort danach stoppen, wollen wir ihn kontinuierlich ausführen, ohne zu viele Terminalfenster oder -tabs geöffnet zu haben.

Die Symfony CLI kann solche Hintergrundbefehle oder Worker verwalten, indem Du das Daemon-Flag (``-d``) zusätzlich zum ``run``-Befehl verwendest.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Führe den Message Consumer erneut aus, aber schiebe ihn in den Hintergrund:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

Die ``--watch``-Option teilt Symfony mit, dass der Befehl neu gestartet werden muss, wenn Dateien in den Verzeichnissen ``config/``, ``src/``, ``templates/`` oder ``vendor/`` verändert werden.

.. note::

    Verwende nicht ``-vv``, da Du sonst in ``server:log`` doppelte Meldungen erhalten würdest (Log- und Konsolenmeldungen).

Wenn der Consumer aus irgendeinem Grund nicht mehr funktioniert (Speicherlimit, Fehler, ...), wird er automatisch neugestartet. Und wenn der Consumer zu schnell versagt, gibt die Symfony CLI auf.

.. index::
    single: Symfony CLI;server:log

Logs werden von ``symfony server:log`` mit allen anderen Logs, die von PHP, dem Webserver und der Anwendung stammen, gesammelt:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Verwende den ``server:status``-Befehl, um alle für das aktuelle Projekt verwalteten Worker aus dem Hintergrund aufzulisten:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Um einen Worker zu stoppen, stoppe den Webserver oder beende die PID, die durch den ``server:status``-Befehl gegeben wird:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Fehlgeschlagene Messages erneut verarbeiten
-------------------------------------------

Was passiert, wenn die Datenbank während des Verarbeitens einer Message ausgefallen ist? Es gibt keine Auswirkungen für Personen, die Kommentare abgeben, aber die Message schlägt fehl und Spam wird nicht überprüft.

Der Messenger hat einen Wiederholungsmechanismus, wenn beim Verarbeiten einer Message ein Fehler auftritt:

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

Wenn beim Verarbeiten einer Message ein Problem auftritt, wird der Consumer es dreimal erneut probieren, bevor er aufgibt. Aber anstatt die Message zu verwerfen, wird sie in einem dauerhafteren Speicher, der ``failed``-Queue, gespeichert, die eine andere Datenbank-Tabelle verwendet.

Überprüfe fehlgeschlagene Messages und verarbeite sie mit den folgenden Befehlen erneut:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Worker bei Upsun ausführen
---------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Um Messages von PostgreSQL zu bearbeiten, müssen wir den ``messenger:consume``-Befehl kontinuierlich ausführen. Bei Upsun ist dies die Rolle eines *Workers*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Wie Symfony CLI verwaltet Upsun Neustarts und Logs.

.. index::
    single: Symfony CLI;cloud:log

Um Logs für einen Worker zu erhalten, verwende:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Weiterführendes

    * `SymfonyCasts Messenger Tutorial`_;

    * Die `Enterprise Service Bus-Architektur`_ und das `CQRS-Pattern`_;

    * Die `Symfony Messenger Dokumentation`_;

.. _`SymfonyCasts Messenger Tutorial`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise Service Bus-Architektur`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS-Pattern`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony Messenger Dokumentation`: https://symfony.com/doc/current/messenger.html
