Volviéndonos asíncronos
=========================

.. index::
    single: Async

La comprobación del correo no deseado durante la gestión del envío del formulario puede dar lugar a algunos problemas. Si el modelo de IA se vuelve lento, nuestro sitio web también lo será para los usuarios. Pero lo que es peor, si tenemos un timeout o si el modelo no está disponible, podemos perder comentarios.

Lo ideal sería que almacenáramos los datos enviados sin publicarlos y devolviéramos inmediatamente una respuesta. La comprobación de spam se puede hacer de forma separada.

Marcando comentarios
--------------------

.. index::
    single: Command;make:entity

Tenemos que introducir un ``state`` para los comentarios: ``submitted``, ``spam`` y ``published``.

Agrega la propiedad ``state`` a la clase ``Comment``:

.. code-block:: terminal
    :class: answers(state||string||255||no)

    $ symfony console make:entity Comment

.. index::
    single: Attributes;ORM\\Column

También debemos asegurarnos de que, por defecto, el valor de ``state`` sea ``submitted``:

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

Crea una migración de base de datos:

.. code-block:: terminal

    $ symfony console make:migration

Modifica la migración para actualizar todos los comentarios existentes para que sean ``published`` de forma predeterminada:

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

Migra la base de datos:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Actualiza la lógica de visualización para evitar que aparezcan comentarios no publicados en el frontend:

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

Actualiza la configuración de EasyAdmin para poder ver el estado del comentario:

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

No olvides también actualizar las factorías de prueba: los comentarios creados por ``CommentFactory`` deben estar publicados por defecto para que aparezcan en las páginas de las conferencias (una prueba siempre puede sobrescribir el estado cuando necesite un comentario moderado):

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

Para las pruebas del controlador, simula la validación:

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

A partir de una prueba de PHPUnit, puedes obtener cualquier servicio del contenedor a través de ``self::getContainer()->get()``; también da acceso a servicios no públicos.

Entendiendo Messenger
---------------------

.. index::
    single: Messenger
    single: Components;Messenger

El componente Messenger es el encargado de la gestión de código asíncrono cuando usamos Symfony:

.. code-block:: terminal

    $ symfony composer req doctrine-messenger

Cuando alguna lógica deba ser ejecutada de forma asíncrona, se envía un mensaje (*message*) a un bus de mensajería (*message bus*). El bus almacena el mensaje en una cola (*queue*) y vuelve inmediatamente para permitir que el flujo de operaciones se reanude lo más rápido posible.

Un *consumidor* se ejecuta continuamente en segundo plano para leer nuevos mensajes en la cola y ejecutar la lógica asociada. El consumidor puede ejecutarse en el mismo servidor que la aplicación web o en un servidor separado.

Es muy similar a la forma en que se manejan las peticiones HTTP, excepto que no tenemos respuestas.

Creando un manejador de mensajes (*Message handler*)
----------------------------------------------------

Un mensaje es una clase de objeto de datos que no debe contener ninguna lógica. Será serializado para ser almacenado en una cola, por lo que sólo se almacenarán datos serializables "simples".

Crea la clase ``CommentMessage``:

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

En el mundo de Messenger, no tenemos controladores, sino manejadores de mensajes.

Crea una clase ``CommentMessageHandler`` bajo un nuevo espacio de nombres ``App\MessageHandler`` que sepa cómo manejar los mensajes ``CommentMessage``:

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

``AsMessageHandler`` ayuda a Symfony a auto-registrar y auto-configurar la clase como un manejador de Messenger. Por convención, la lógica de un manejador vive en un método llamado ``__invoke()``. El *type-hint* ``CommentMessage`` en el único argumento de este método le dice a Messenger qué clase manejará.

Actualiza el controlador para utilizar el nuevo sistema:

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

En lugar de depender del Spam Checker, ahora enviamos un mensaje al bus. El manejador entonces decide qué hacer con él.

Hemos logrado algo inesperado. Hemos desacoplado nuestro controlador del Spam Checker y hemos movido la lógica a una nueva clase, el manejador. Es un caso de uso perfecto para el bus. Prueba el código, funciona. Todo se sigue haciendo sincrónicamente, pero el código probablemente ya sea "mejor".

Volviéndonos asíncronos de verdad
-----------------------------------

Por defecto, los manejadores son llamados sincrónicamente. Para ser asíncronos, es necesario configurar explícitamente qué cola utilizar para cada manejador en el fichero de configuración ``config/packages/messenger.yaml``:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/messenger.yaml
    +++ w/config/packages/messenger.yaml
    @@ -26,4 +26,4 @@ framework:
                 Symfony\Component\Notifier\Message\SmsMessage: async

                 # Route your messages to the transports
    -            # 'App\Message\YourMessage': async
    +            App\Message\CommentMessage: async

La configuración le dice al bus que envíe instancias de ``App\Message\CommentMessage`` a la cola ``async``, que está definida por un DSN (``MESSENGER_TRANSPORT_DSN``), que apunta a Doctrine como está configurado en ``.env``. En pocas palabras, estamos usando PostgreSQL como una cola para nuestros mensajes.

.. tip::

    Por detrás, Symfony utiliza el sistema de publicación, de alto rendimiento, escalable y transaccional de PostgreSQL (``LISTEN``/``NOTIFY``). También puedes leer el capítulo RabbitMQ si deseas utilizarlo en lugar de PostgreSQL como intermediario de mensajes.

Consumiendo mensajes
--------------------

Si intentas enviar un nuevo comentario, ya no se llamará al verificador de spam. Agrega una llamada ``error_log()`` en el método ``getSpamScore()`` para confirmar. En su lugar, un mensaje está esperando en la cola, listo para ser consumido por algunos procesos.

.. index::
    single: Command;messenger:consume

Como te puedes imaginar, Symfony viene con un comando de consumidor. Ejecútalo ahora:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:consume async -vv

Debe consumir inmediatamente el mensaje despachado para el comentario enviado:

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

La actividad del consumidor de mensajes se registra, pero obtendrás información en tiempo real en la consola si le pasas el parámetro ``-vv``. Incluso deberías ser capaz de detectar la llamada a la API de OpenAI.

Para detener al consumidor, pulsa ``Ctrl+C``.

Ejecutando *workers* en segundo plano
-------------------------------------

En lugar de lanzar al consumidor cada vez que publicamos un comentario y detenerlo inmediatamente después, queremos ejecutarlo continuamente sin tener demasiadas ventanas de terminal o pestañas abiertas.

El comando Symfony puede administrar dichos comandos en segundo plano o *workers* (trabajadores) usando el parámetro de *daemon* (``-d``) en el comando ``run``.

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;run -d --watch

Ejecuta de nuevo el consumidor del mensaje, pero envíalo en segundo plano:

.. code-block:: terminal

    $ symfony run -d --watch=config,src,templates,vendor/composer/installed.json symfony console messenger:consume async -vv

La opción ``--watch`` le dice a Symfony que el comando debe ser reiniciado siempre que haya un cambio en el sistema de archivos en los directorios ``config/``, ``src/``, ``templates/`` o ``vendor/``.

.. note::

    No uses ``-vv`` ya que obtendrías mensajes duplicados en ``server:log`` (los mensajes registrados y los mensajes de consola).

Si el consumidor dejara de trabajar por alguna razón (por quedarse sin memoria, por un fallo...), se reiniciará automáticamente. Y si el consumidor falla demasiado rápido, el comando Symfony desistirá de reiniciarlo.

.. index::
    single: Symfony CLI;server:log

Usando ``symfony server:log`` los registros generados se unirán a todos los demás registros procedentes de PHP, el servidor web y la aplicación:

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

.. index::
    single: Command;messenger:consume
    single: Symfony CLI;server:status

Utiliza el comando ``server:status`` para listar todos los *workers* en segundo plano pertenecientes al proyecto actual:

.. code-block:: terminal
    :class: ignore

    $ symfony server:status

    Web server listening on https://127.0.0.1:8000
      Command symfony console messenger:consume async running with PID 15774 (watching config/, src/, templates/)

Para detener a un *worker*, detén el servidor web o mata el proceso que tiene el PID que se muestra con el comando ``server:status``:

.. code-block:: terminal
    :class: ignore

    $ kill 15774

Reintentando cuando los mensajes fallan
---------------------------------------

¿Qué pasa si la base de datos se cae mientras se consume un mensaje? Las personas que envían comentarios no notarán nada, pero el mensaje falla y el spam no se controla.

Messenger tiene un mecanismo de reintento para cuando ocurre una excepción mientras se maneja un mensaje:

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

Si ocurre un problema mientras se maneja un mensaje, el consumidor volverá a intentarlo 3 veces antes de darse por vencido. Pero en lugar de descartar el mensaje, lo almacenará permanentemente en la cola ``failed``, que utiliza otra tabla de la base de datos.

Inspecciona los mensajes fallidos y vuelve a intentarlo mediante los siguientes comandos:

.. code-block:: terminal
    :class: ignore

    $ symfony console messenger:failed:show

    $ symfony console messenger:failed:retry

Ejecutando *workers* en Upsun
------------------------------------

.. index::
    single: Upsun;Workers
    single: Workers

Para consumir mensajes de PostgreSQL, necesitamos ejecutar el comando ``messenger:consume`` continuamente. En Upsun, este es el rol de un *worker*:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 1,5
    :class: ignore

    workers:
        messenger:
            commands:
                # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
                start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async

Al igual que hace el comando Symfony, Upsun también gestiona los reinicios y los registros.

.. index::
    single: Symfony CLI;cloud:log

Para obtener *logs* de un *worker*, usa:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --worker=messages all

.. sidebar:: Yendo más allá

    * `Tutorial de SymfonyCasts Messenger`_ ;

    * La arquitectura del `Enterprise service bus`_ y el `patrón CQRS`_ ;

    * La `documentación de Symfony Messenger`_ ;

.. _`Tutorial de SymfonyCasts Messenger`: https://symfonycasts.com/screencast/messenger
.. _`Enterprise service bus`: https://en.wikipedia.org/wiki/Enterprise_service_bus
.. _`patrón CQRS`: https://martinfowler.com/bliki/CQRS.html
.. _`documentación de Symfony Messenger`: https://symfony.com/doc/current/messenger.html
