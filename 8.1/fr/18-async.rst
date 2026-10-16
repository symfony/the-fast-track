Faire de l'asynchrone
=====================

.. index::
    single: Async

Vérifier la présence de spam pendant le traitement de la soumission du formulaire peut entraîner certains problèmes. Si le modèle d'IA devient lent, notre site web sera également lent pour les internautes. Mais pire encore, si nous atteignons le délai d'attente maximal ou si le modèle n'est pas disponible, nous pourrions perdre des commentaires.

Idéalement, nous devrions stocker les données soumises, sans les publier, et renvoyer une réponse immédiatement. La vérification du spam pourra être faite par la suite.

Marquer les commentaires
------------------------

.. index::
    single: Command;make:entity

Nous avons besoin d'introduire un état (``state``) pour les commentaires : ``submitted``, ``spam`` et ``published``.

Ajoutez la propriété ``state`` à la classe ``Comment`` :

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

Nous devrions également nous assurer que, par défaut, le paramètre ``state`` est initialisé avec la valeur ``submitted`` :

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

Créez une migration de base de données :

.. code-block:: terminal

    $ symfony console make:migration

Modifiez la migration pour mettre à jour tous les commentaires existants comme étant ``published`` par défaut :

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

Migrez la base de données :

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Modifiez la logique d'affichage pour éviter que des commentaires non publiés n'apparaissent sur le site :

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

Modifiez la configuration d'EasyAdmin pour voir l'état du commentaire :

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

N'oubliez pas de modifier également les factories de test : les commentaires créés par ``CommentFactory`` doivent être publiés par défaut afin d'apparaître sur les pages des conférences (un test peut toujours surcharger l'état lorsqu'il a besoin d'un commentaire en cours de modération) :

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

Pour les tests du contrôleur, simulez la validation :

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

À partir d'un test PHPUnit, vous pouvez obtenir n'importe quel service depuis le conteneur grâce à ``self::getContainer()->get()`` ; il donne également accès aux services non publics.

Comprendre Messenger
--------------------

.. index::
    single: Messenger
    single: Components;Messenger

La gestion du code asynchrone avec Symfony est faite par le composant Messenger :

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Lorsqu'une action doit être exécutée de manière asynchrone, envoyez un *message* à un *messenger bus*. Le bus stocke le message dans une *file d'attente* et rend immédiatement la main pour permettre au flux des opérations de reprendre aussi vite que possible.

Un *consumer* s'exécute continuellement en arrière-plan pour lire les nouveaux messages dans la file d'attente et exécuter la logique associée. Le consumer peut s'exécuter sur le même serveur que l'application web, ou sur un serveur séparé.

C'est très similaire à la façon dont les requêtes HTTP sont traitées, sauf que nous n'avons pas de réponse.

Coder un gestionnaire de messages
---------------------------------

Un message est une classe de données (*data object*), qui ne doit contenir aucune logique. Il sera sérialisé pour être stocké dans une file d'attente, donc ne stockez que des données "simples" et sérialisables.

Créez la classe ``CommentMessage`` :

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

Dans le monde de Messenger, nous n'avons pas de contrôleurs, mais des gestionnaires de messages.

Sous un nouveau namespace ``App\MessageHandler``, créez une classe ``CommentMessageHandler`` qui saura comment gérer les messages ``CommentMessage`` :

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

``AsMessageHandler`` aide Symfony à enregistrer et à configurer automatiquement la classe en tant que gestionnaire Messenger. Par convention, la logique d'un gestionnaire réside dans une méthode appelée ``__invoke()``. Le type ``CommentMessage`` précisé en tant qu'argument unique de cette méthode indique à Messenger quelle classe elle va gérer.

Modifiez le contrôleur pour utiliser le nouveau système :

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

Au lieu de dépendre du SpamChecker, nous envoyons maintenant un message dans le bus. Le gestionnaire décide alors ce qu'il en fait.

Nous avons fait quelque chose que nous n'avions pas prévu. Nous avons découplé notre contrôleur du vérificateur de spam, et déplacé la logique vers une nouvelle classe, le gestionnaire. C'est un cas d'utilisation parfait pour le bus. Testez le code, il fonctionne. Tout se fait encore de manière synchrone, mais le code est probablement déjà "mieux".

Faire vraiment de l'asynchrone
------------------------------

Par défaut, les gestionnaires sont appelés de manière synchrone. Pour les rendre asynchrone, vous devez configurer explicitement la file d'attente à utiliser pour chaque gestionnaire dans le fichier de configuration ``config/packages/messenger.yaml`` :

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

La configuration indique au bus d'envoyer les instances de ``App\Message\CommentMessage`` à la file d'attente ``async``, qui est définie par un DSN (``MESSENGER_TRANSPORT_DSN``), qui pointe vers Doctrine tel que défini dans le fichier ``.env``. En clair, nous utilisons PostgreSQL comme file d'attente pour nos messages.

.. tip::

    En coulisses, Symfony utilise le système pub/sub intégré, performant, dimensionnable (``LISTEN``/``NOTIFY``). Vous pouvez aussi lire le chapitre sur RabbitMQ si vous voulez l'utiliser à la place de PostgreSQL comme gestionnaire de messages.

Consommer des messages
----------------------

Si vous essayez de soumettre un nouveau commentaire, le vérificateur de spam ne sera plus appelé. Ajoutez un appel à la fonction ``error_log()`` dans la méthode ``getSpamScore()`` pour le confirmer. Au lieu d'avoir un nouveau commentaire, un message est en attente dans la file d'attente, prêt à être consommé par d'autres processus.

.. index::
    single: Command;messenger:consume

Comme vous pouvez l'imaginer, Symfony est livré avec une commande pour consommer les messages. Exécutez-la maintenant :

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Cette commande devrait immédiatement consommer le message envoyé pour le commentaire soumis :

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

L'activité du consumer de messages est enregistrée dans les logs, mais vous pouvez avoir un affichage instantané dans la console en passant l'option ``-vv``. Vous devriez même voir l'appel vers l'API d'OpenAI.

Pour arrêter le consumer, appuyez sur ``Ctrl+C``.

Lancer des *workers* en arrière-plan
-------------------------------------

Au lieu de lancer le consumer à chaque fois que nous publions un commentaire et de l'arrêter immédiatement après, nous voulons l'exécuter en continu sans avoir trop de fenêtres ou d'onglets du terminal ouverts.

La commande ``symfony`` peut gérer des commandes en tâche de fond ou des workers en utilisant l'option daemon (``-d``) sur la commande ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Exécutez à nouveau le consumer du message, mais en tâche de fond :

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

L'option ``--watch`` indique à Symfony que la commande doit être redémarrée chaque fois qu'il y a un changement dans un des fichiers des répertoires ``config/``, ``vendor/``, ``src/`` ou ``templates/``.

.. note::

    N'utilisez pas ``-vv`` pour éviter les doublons dans ``server:log`` (messages enregistrés et messages de la console).

Si le consumer cesse de fonctionner pour une raison quelconque (limite de mémoire, bogue, etc.), il sera redémarré automatiquement. Et s'il tombe en panne trop rapidement, la commande ``symfony`` s'arrêtera.

.. index::
    single: Symfony CLI;server:log

Les logs sont diffusés en continu par la commande ``symfony server:log``, en même temps que ceux de PHP, du serveur web et de l'application :

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Utilisez la commande ``server:status`` pour lister tous les workers en arrière-plan gérés pour le projet en cours :

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Pour arrêter un worker, arrêtez le serveur web ou tuez le PID (identifiant du processus) donné par la commande ``server:status`` :

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Renvoyer des messages ayant échoué
------------------------------------

Que faire si la base de données est en panne alors qu'un message est en train d'être consommé ? Il n'y a aucun impact pour les personnes qui soumettent des commentaires, mais le message échoue et le spam n'est pas vérifié.

Messenger dispose d'un mécanisme de relance lorsqu'une exception se produit lors du traitement d'un message :

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

Si un problème survient lors de la manipulation d'un message, le consumer réessaiera 3 fois avant d'abandonner. Mais au lieu de jeter le message, il le stockera indéfiniment dans la file d'attente ``failed``, qui utilise une autre table de la base de données.

Inspectez les messages ayant échoué et relancez-les à l'aide des commandes suivantes :

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Exécuter des workers sur Upsun
-------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Pour consommer les messages de PostgreSQL, nous devons exécuter la commande ``messenger:consume`` en continu. Sur Upsun, c'est le rôle d'un *worker* :

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Comme pour la commande ``symfony``, Upsun gère les redémarrages et les logs.

.. index::
    single: Symfony CLI;cloud:log

Pour obtenir les logs d'un worker, utilisez :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Aller plus loin

    * `Tutoriel SymfonyCasts sur Messenger`_ ;

    * L'architecture de l'`Enterprise service bus`_ et le `modèle CQRS`_ ;

    * La `documentation de Symfony Messenger`_ ;

.. _`Tutoriel SymfonyCasts sur Messenger`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`modèle CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`documentation de Symfony Messenger`: https://symfony.com/doc/current/messenger.html
