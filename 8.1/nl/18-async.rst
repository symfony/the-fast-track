Async gaan
==========

.. index::
    single: Async

Het controleren op spam tijdens afhandelen van het formulier kan tot problemen leiden. Als het AI-model traag wordt, dan zal onze website ook traag worden voor gebruikers. Maar erger nog, als er een time-out optreedt of als het model niet beschikbaar is, dan kunnen we reacties kwijtraken.

Idealiter slaan we de ingediende gegevens op zonder ze te publiceren en sturen we onmiddellijk een response terug. Het controleren op spam kan dan later gebeuren.

Reacties markeren
-----------------

.. index::
    single: Command;make:entity

We moeten een ``state`` voor reacties introduceren: ``submitted``, ``spam`` en ``published``.

Voeg de ``state`` property toe aan de ``Comment`` class:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

We moeten er ook voor zorgen dat de ``state`` standaard ``submitted`` is:

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

Maak een databasemigratie aan:

.. code-block:: terminal

    $ symfony console make:migration

Maak de migratie zo dat bestaande reacties standaard ``published`` zijn:

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

Migreer de database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Werk de weergavelogica bij om te voorkomen dat niet-gepubliceerde reacties in de frontend verschijnen:

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

Update de EasyAdmin-configuratie om de status van de reactie te kunnen zien:

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

Vergeet niet om ook de test-factories bij te werken: reacties die door ``CommentFactory`` worden aangemaakt, moeten standaard gepubliceerd zijn zodat ze op de conferentiepagina's verschijnen (een test kan de state altijd overschrijven wanneer een gemodereerde reactie nodig is):

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

Simuleer de validatie voor de controller tests:

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

Vanuit een PHPUnit-test kan je elke service uit de container halen via ``self::getContainer()->get()``; dit geeft ook toegang tot niet-publieke services.

De Messenger begrijpen
----------------------

.. index::
    single: Messenger
    single: Components;Messenger

Het beheren van asynchrone code met Symfony is de taak van de Messenger-component:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Wanneer logica asynchroon moet worden uitgevoerd, stuur dan een *message* naar een *messenger bus*. De bus voegt het bericht toe aan een *queue* en keert onmiddellijk terug om de flow van operaties zo snel mogelijk te laten hervatten.

Een *consumer* loopt continu op de achtergrond om nieuwe berichten in de queue te lezen en de bijbehorende logica uit te voeren. De consumer kan op dezelfde server draaien als de webapplicatie of op een aparte server.

Het lijkt sterk op de manier waarop HTTP-request worden behandeld, behalve dat we geen responses hebben.

Bouwen van een Message Handler
------------------------------

Een bericht is een data object class die geen logica mag bevatten en die wordt geserialiseerd om in een queue te worden opgeslagen, dus voeg er alleen "simpele" serialiseerbare gegevens aan toe.

Maak de ``CommentMessage`` class:

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

In de Messenger wereld hebben we geen controllers, maar message handlers.

Maak een ``CommentMessageHandler`` class aan onder een nieuwe ``App\MessageHandler`` namespace die weet hoe een ``CommentMessage`` verwerkt moet worden:

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

``AsMessageHandler`` helpt Symfony alleen bij het automatisch registreren en automatisch configureren van de class als een Messenger handler. Volgens de conventie hoort logica van een handler in de ``__invoke()`` methode te staan. De ``CommentMessage`` type hint op het enige argument van deze methode, vertelt Messenger welke class er afgehandeld moet worden.

Werk de controller bij om ervoor te zorgen dat deze het nieuwe systeem gebruikt:

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

In plaats van afhankelijk te zijn van de Spam Checker, sturen we nu een message naar de bus. De handler beslist dan wat er mee te doen.

We hebben iets onverwachts bereikt. We hebben onze controller losgekoppeld van de Spam Checker en de logica verplaatst naar een nieuwe class, de handler. Dit is een perfecte use case voor de bus. Test de code, het werkt. Alles wordt nog steeds synchroon gedaan, maar de code is waarschijnlijk al "beter".

Echt async werken
-----------------

Standaard worden handlers synchroon aangeroepen. Om async te gaan, moet je expliciet configureren welke queue voor welke handler gebruikt moet worden. Dit kan in het configuratiebestand ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

De configuratie vertelt de bus om instanties van ``App\Message\CommentMessage`` naar de ``async`` queue te sturen, die wordt gedefinieerd door een DSN (``MESSENGER_TRANSPORT_DSN``), die verwijst naar Doctrine zoals geconfigureerd in ``.env``. In gewoon Nederlands gebruiken we PostgreSQL als wachtrij voor onze berichten.

.. tip::

    Achter de schermen gebruikt Symfony het ingebouwde, efficiënte, schaalbare en transactionele pub/subsysteem van PostgreSQL (``LISTEN``/``NOTIFY``). Je kunt ook het RabbitMQ-hoofdstuk lezen als je het in plaats van PostgreSQL als message-broker wilt gebruiken.

Messages consumeren
-------------------

Als je probeert een nieuwe reactie toe te voegen, dan zal de spamchecker niet meer aangeroepen worden. Voeg een ``error_log()`` aanroep toe in de ``getSpamScore()`` methode om dit te bevestigen. In plaats daarvan wacht er een bericht in de queue, klaar om door bepaalde processen te worden geconsumeerd.

.. index::
    single: Command;messenger:consume

Zoals je je kan voorstellen, komt Symfony met een consumer command. Voer dat nu uit:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Het bericht van de ingevoerde reactie zou direct geconsumeerd moeten worden:

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

De activiteit van de message consumer is gelogd, maar je krijgt direct feedback in de console door de ``-vv`` flag mee te geven. Je zal zelfs de OpenAI API call zien voorbijkomen.

Gebruik ``Ctrl+C`` om de consumer te stoppen.

Workers op de achtergrond laten draaien
---------------------------------------

In plaats van de consumer iedere keer te starten en te stoppen als er een reactie wordt geplaatst, willen we dat het proces continu op de achtergrond draait, zonder dat we te veel terminal windows of tabbladen open hebben.

De Symfony CLI kan dit soort background commando's of workers managen door de daemon flag (``-d``) te gebruiken bij het ``run`` commando.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Voer de message consumer opnieuw uit, maar laat hem dit keer in de achtergrond draaien:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

De ``--watch`` optie vertelt Symfony dat het commando opnieuw moet worden gestart wanneer er een wijziging op het bestandssysteem in de ``config/``, ``src/``, ``templates/``, of ``vendor/`` directories optreedt.

.. note::

    Gebruik ``-vv`` niet, anders zie je dubbele berichten in ``server:log`` (gelogde berichten en console berichten).

Als de consumer om de een of andere reden stopt met werken (geheugenlimiet, bug, ....), dan wordt deze automatisch opnieuw opgestart. En als de consumer te snel faalt, dan geeft de Symfony CLI het op.

.. index::
    single: Symfony CLI;server:log

De logs worden gestreamd via ``symfony server:log`` samen met alle andere logs die afkomstig zijn van PHP, de webserver en de applicatie:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Gebruik het ``server:status`` commando om een lijst te maken van alle workers die op de achtergrond draaien voor het huidige project:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Om een worker te stoppen, stop je de webserver of kill je de PID die door het ``server:status`` commando wordt teruggegeven:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Mislukte berichten opnieuw proberen
-----------------------------------

En wat als de database down is terwijl er een bericht wordt geconsumeerd? Er is geen impact voor mensen die een reactie geven, maar het bericht mislukt en spam wordt niet gecontroleerd.

Messenger heeft een retry mechanisme voor het geval er een exception optreedt tijdens de afhandeling van een bericht:

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

Als er zich een probleem voordoet bij het afhandelen van een bericht, dan zal de consumer dit 3 maal opnieuw proberen voordat deze het opgeeft. Maar in plaats van het bericht te verwijderen, zal het bericht permanent worden opgeslagen in de ``failed`` queue, die gebruik maakt van een andere database tabel.

Inspecteer de mislukte berichten en probeer ze opnieuw aan te bieden via de volgende commando's:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Workers draaien op Upsun
------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Om berichten van PostgreSQL te consumeren, moeten we het ``messenger:consume`` commando continu uitvoeren. Op Upsun is dit de rol van een *worker*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Net als bij de Symfony CLI, beheert Upsun de restarts en logs.

.. index::
    single: Symfony CLI;cloud:log

Om logs voor een worker te krijgen, gebruik:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Verder gaan

    * `SymfonyCasts Messenger tutorial`_;

    * De `Enterprise service bus`_ architectuur en het `CQRS patroon`_;

    * De `Symfony Messenger documentatie`_;

.. _`SymfonyCasts Messenger tutorial`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`CQRS patroon`: https://martinfowler.com/bliki/CQRS.html
.. _`Symfony Messenger documentatie`: https://symfony.com/doc/current/messenger.html
