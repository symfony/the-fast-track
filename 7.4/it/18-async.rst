Esecuzione asincrona
====================

.. index::
    single: Async

Controllare la presenza di spam durante la gestione dell'invio del form potrebbe portare ad alcuni problemi. Se le API di Akismet diventano lente, il nostro sito web lo sarà anche per gli utenti. Ma peggio ancora, se si verifica un timeout o se le API di Akismet sono temporaneamente non disponibili, potremmo perdere dei commenti.

Idealmente, dovremmo salvare i dati inviati senza pubblicarli e restituire immediatamente una risposta. Lo spam può essere controllato in un secondo momento.

Marcare i commenti
------------------

.. index::
    single: Command;make:entity

Dobbiamo introdurre uno stato (``state``) per i commenti: ``submitted``, ``spam`` e ``published``.

Aggiungiamo la proprietà ``state`` alla classe ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Dovremmo anche assicurarci che il valore predefinito di ``state`` sia ``submitted``:

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

Creare una migration per il database:

.. code-block:: terminal

    $ symfony console make:migration

Modificare la migration per aggiornare tutti i commenti esistenti, impostando il loro stato predefinito a ``published``:

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

Migrazione del database:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Aggiornare la logica di visualizzazione per evitare che i commenti non pubblicati siano visibili sul frontend:

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

Aggiornare la configurazione di EasyAdmin per poter vedere lo stato del commento:

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

Non dimentichiamo di aggiornare anche i test impostando lo ``state`` nelle fixture:

.. code-block:: diff
    :caption: patch_file

    --- i/src/DataFixtures/AppFixtures.php
    +++ w/src/DataFixtures/AppFixtures.php
    @@ -35,8 +35,16 @@ class AppFixtures extends Fixture
             $comment1->setAuthor('Fabien');
             $comment1->setEmail('fabien@example.com');
             $comment1->setText('This was a great conference.');
    +        $comment1->setState('published');
             $manager->persist($comment1);

    +        $comment2 = new Comment();
    +        $comment2->setConference($amsterdam);
    +        $comment2->setAuthor('Lucas');
    +        $comment2->setEmail('lucas@example.com');
    +        $comment2->setText('I think this one is going to be moderated.');
    +        $manager->persist($comment2);
    +
             $admin = new Admin();
             $admin->setRoles(['ROLE_ADMIN']);
             $admin->setUsername('admin');

.. index::
    single: Test;Container
    single: Container;Test

Per i test dei controller, simulare la validazione:

.. code-block:: diff
    :caption: patch_file

    --- i/tests/Controller/ConferenceControllerTest.php
    +++ w/tests/Controller/ConferenceControllerTest.php
    @@ -2,6 +2,8 @@

     namespace App\Tests\Controller;

    +use App\Repository\CommentRepository;
    +use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

     class ConferenceControllerTest extends WebTestCase
    @@ -22,10 +24,16 @@ class ConferenceControllerTest extends WebTestCase
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

In un test PHPUnit, è possibile ottenere qualsiasi servizio dal container tramite ``self::$container->get()``; oltretutto offre anche accesso ai servizi non pubblici.

Comprendere Messenger
---------------------

.. index::
    single: Messenger
    single: Components;Messenger

Gestire codice asincrono con Symfony è il compito del componente Messenger:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Quando una logica deve essere eseguita in maniera asincrona, inviare un *messaggio* ad un *messenger bus*. Questo memorizza il messaggio in una *coda* e restituisce immediatamente il controllo per far ripartire il flusso delle operazioni il più velocemente possibile.

Un *consumer* è eseguito costantemente in background in modo da leggere nuovi messaggi dalla coda ed eseguire la logica associata. Un consumer può essere eseguito sullo stesso server dell'applicazione web oppure su uno separato.

È molto simile al modo in cui vengono gestite le richieste HTTP, tranne per il fatto che non abbiamo risposte.

Scrivere un message handler
---------------------------

Un messaggio è un oggetto che non dovrebbe contenere alcuna logica, in quanto sarà serializzato per essere memorizzato in una coda. Pertanto utilizzate solo dati "semplici" e serializzabili.

Creare la classe ``CommentMessage``:

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

Nel mondo di Messenger non abbiamo controller, ma *message handler* (gestori di messaggi).

All'interno di un nuovo namespace chiamato ``App\MessageHandler``, creare la classe ``CommentMessageHandler``, che saprà gestire i messaggi di tipo ``CommentMessage``:

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

        public function __invoke(CommentMessage $message)
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

``AsMessageHandler`` aiuta Symfony ad auto-registrare e auto-configurare la classe come Messenger handler. Per convenzione, la logica di gestione risiede in un metodo chiamato ``__invoke()``. Il tipo ``CommentMessage`` sul parametro di questo metodo dice a Messenger quale classe sarà in grado di gestire.

Aggiornare il controller per utilizzare il nuovo sistema:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -5,21 +5,23 @@ namespace App\Controller;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use App\Form\CommentType;
    +use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    -use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
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
             #[MapEntity(mapping: ['slug' => 'slug'])]
             Conference $conference,
             CommentRepository $commentRepository,
    -        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
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

Invece di dipendere dallo Spam Checker, ora inviamo un messaggio alla coda, e il gestore (handler) in un secondo momento deciderà cosa farne.

Abbiamo ottenuto qualcosa di inaspettato. Abbiamo disaccoppiato il nostro controller dallo Spam Checker e spostato la logica in una nuova classe: l'handler (il nostro gestore). Questo infatti è un perfetto caso d'uso per una coda. Testiamo il codice. Tutto è ancora eseguito in maniera sincrona, ma il codice è probabilmente già "migliore".

Eseguiamolo in maniera asincrona
--------------------------------

Per impostazione predefinita, gli handler (i gestori) sono chiamati in modo sincrono. Per essere eseguiti in maniera asincrona, è necessario configurare esplicitamente la coda da usare per ognuno di essi, nel file di configurazione ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

La configurazione indica al bus di inviare istanze di tipo ``App\Message\CommentMessage`` nella coda di tipo ``async``, definita da un DSN (``MESSENGER_TRANSPORT_DSN``), che punta a Doctrine come configurato in ``.env``. In linguaggio naturale diremmo che stiamo usando PostgreSQL come coda per i nostri messaggi.

.. tip::

    Dietro le quinte, Symfony utilizza il sistema interno pub/sub (``LISTEN``/``NOTIFY``) di PostgreSQL, che è performante, scalabile e transazionale. Potete leggere il capitolo RabbitMQ se volete utilizzare Rabbit come message broker invece di PostgreSQL.

Consumare i messaggi
--------------------

Se si tenta di inviare un nuovo commento, lo Spam Checker non verrà più chiamato. Chiamare ``error_log()`` nel metodo ``getSpamScore()`` per averne conferma. Se controlliamo, un messaggio è invece in attesa nella coda, pronto per essere consumato da qualche processo.

.. index::
    single: Command;messenger:consume

In Symfony è presente un comando per gestire i consumer. Eseguiamolo:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Dovrebbe consumare immediatamente il messaggio inviato, grazie al commento inviato:

.. code-block:: text
    :class: ignore

     [OK] Consuming messages from transports "async".

     // The worker will automatically exit once it has received a stop signal via the messenger:stop-workers command.

     // Quit the worker with CONTROL-C.

    11:30:20 INFO      [messenger] Received message App\Message\CommentMessage ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]
    11:30:20 INFO      [http_client] Request: "POST https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [http_client] Response: "200 https://80cea32be1f6.rest.akismet.com/1.1/comment-check"
    11:30:20 INFO      [messenger] Message App\Message\CommentMessage handled by App\MessageHandler\CommentMessageHandler::__invoke ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage","handler" => "App\MessageHandler\CommentMessageHandler::__invoke"]
    11:30:20 INFO      [messenger] App\Message\CommentMessage was handled successfully (acknowledging to transport). ["message" => App\Message\CommentMessage^ { …},"class" => "App\Message\CommentMessage"]

L'attività di consumo dei messaggi dalla coda viene salvata nei log, ma è possibile ottenere un feedback immediato in console aggiungendo al comando l'opzione ``-vv``. In questo modo si dovrebbe anche poter vedere la chiamata alle API di Akismet.

Per fermare il consumer premere ``Ctrl+C``.

Esecuzione in background dei worker
-----------------------------------

Invece di eseguire il consumer ogni volta che si pubblica un commento per poi fermarlo subito dopo, vogliamo che sia sempre in esecuzione senza avere troppe finestre del terminale o schede del aperte.

La CLI di Symfony può eseguire questi comandi in background aggiungendo l'opzione demone (``-d``) al comando ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Eseguire di nuovo il consumer, ma questa volta in background:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

L'opzione ``--watch`` dice a Symfony che il comando deve essere riavviato ogni volta che si verifica una modifica al filesystem nelle cartelle ``config/``, ``src/``, ``templates/`` oppure ``vendor/``.

.. note::

    Non utilizzare l'opzione ``-vv``, altrimenti ci saranno messaggi duplicati in ``server:log`` (log dei messaggi e messaggi della console).

Se il consumer smette di funzionare a causa di un errore (memory limit, bug, ecc.), verrà riavviato automaticamente. Invece, nel caso in cui questo smetta di funzionare troppo velocemente, la CLI di Symfony smetterà di riavviarlo.

.. index::
    single: Symfony CLI;server:log

I log possono essere mostrati eseguendo il comando ``symfony server:log`` visualizzando così anche tutti gli altri log provenienti da PHP, server web e applicazione:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Utilizzare il comando ``server:status`` per visualizzare tutti i worker gestiti in background per questo progetto:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Per fermare un worker occorre fermare il server web, oppure eseguire il comando di sistema "kill" seguito dal suo PID, che si può recuperare tramite il comando ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Riprovare con i messaggi falliti
--------------------------------

E se le API di Akismet non fossero disponibili mentre viene consumato un messaggio? Questo non farà alcuna differenza per l'utente che invia un commento, ma il messaggio andrà perso, e non ci sarà alcun controllo sulla presenza di spam.

Messenger ha un meccanismo di "retry" per i casi in cui si verifichi un'eccezione durante la gestione di un messaggio:

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

Se si verifica un errore durante la gestione di un messaggio, il consumer riproverà tre volte prima rinunciare. Ma invece di scartare il messaggio, lo memorizzerà permanentemente nella coda ``failed``, che usa un'altra tabella di database.

Ispezionare i messaggi che sono falliti e provare a gestirli di nuovo con i seguenti comandi:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Eseguire i worker su Upsun
--------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Per consumare i messaggi da PostgreSQL, dobbiamo eseguire il comando ``messenger:consume``. Su Upsun, questo è il ruolo di un *worker*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Come per la CLI di Symfony, Upsun gestisce riavvii e log.

.. index::
    single: Symfony CLI;cloud:log

Per mostrare i log di un worker, utilizzare:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Andare oltre

    * Tutorial SymfonyCasts su `Messenger`_;

    * L'architettura `Enterprise service bus`_ e il `pattern CQRS`_;

    * `Documentazione su Symfony Messenger`_;

.. _`Messenger`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`pattern CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`Documentazione su Symfony Messenger`: https://symfony.com/doc/current/messenger.html
