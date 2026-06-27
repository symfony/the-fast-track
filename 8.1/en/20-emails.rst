Emailing Admins
===============

.. index::
    single: Components;Mailer
    single: Mailer
    single: Emails

To ensure high quality feedback, the admin must moderate all comments. When a comment is in the ``ham`` or ``potential_spam`` state, an *email* should be sent to the admin with two links: one to accept the comment and one to reject it.

Setting an Email for the Admin
------------------------------

To store the admin email, use a container parameter. For demonstration purposes, we also allow it to be set via an environment variable (should not be needed in "real life"):

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -5,6 +5,8 @@
     # https://symfony.com/doc/current/best_practices.html#use-parameters-for-application-configuration
     parameters:
         photo_dir: "%kernel.project_dir%/public/uploads/photos"
    +    default_admin_email: admin@example.com
    +    admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"

     services:
         # default configuration for services in *this* file

An environment variable might be "processed" before being used. Here, we are using the ``default`` processor to fall back to the value of the ``default_admin_email`` parameter if the ``ADMIN_EMAIL`` environment variable does not exist.

Sending a Notification Email
----------------------------

To send an email, you can choose between several ``Email`` class abstractions; from ``Message``, the lowest level, to ``NotificationEmail``, the highest one. You will probably use the ``Email`` class the most, but ``NotificationEmail`` is the perfect choice for internal emails.

In the message handler, let's replace the auto-validation logic:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -7,7 +7,10 @@ use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Psr\Log\LoggerInterface;
    +use Symfony\Bridge\Twig\Mime\NotificationEmail;
    +use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\DependencyInjection\Attribute\Target;
    +use Symfony\Component\Mailer\MailerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -21,6 +24,8 @@ class CommentMessageHandler
             private CommentRepository $commentRepository,
             private MessageBusInterface $bus,
             #[Target('comment')] private WorkflowInterface $workflow,
    +        private MailerInterface $mailer,
    +        #[Autowire('%admin_email%')] private string $adminEmail,
             private ?LoggerInterface $logger = null,
         ) {
         }
    @@ -43,8 +48,13 @@ class CommentMessageHandler
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
             } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    -            $this->workflow->apply($comment, $this->workflow->can($comment, 'publish') ? 'publish' : 'publish_ham');
    -            $this->entityManager->flush();
    +            $this->mailer->send((new NotificationEmail())
    +                ->subject('New comment posted')
    +                ->htmlTemplate('emails/comment_notification.html.twig')
    +                ->from($this->adminEmail)
    +                ->to($this->adminEmail)
    +                ->context(['comment' => $comment])
    +            );
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

The ``MailerInterface`` is the main entry point and allows to ``send()`` emails.

To send an email, we need a sender (the ``From``/``Sender`` header). Instead of setting it explicitly on the Email instance, define it globally:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/mailer.yaml
    +++ w/config/packages/mailer.yaml
    @@ -1,3 +1,5 @@
     framework:
         mailer:
             dsn: '%env(MAILER_DSN)%'
    +        envelope:
    +            sender: "%admin_email%"

Extending the Notification Email Template
-----------------------------------------

.. index::
    single: Twig;extends
    single: Twig;block
    single: Twig;url

The notification email template inherits from the default notification email template that comes with Symfony:

.. code-block:: html+twig
    :caption: templates/emails/comment_notification.html.twig

    {% extends '@email/default/notification/body.html.twig' %}

    {% block content %}
        Author: {{ comment.author }}<br />
        Email: {{ comment.email }}<br />
        State: {{ comment.state }}<br />

        <p>
            {{ comment.text }}
        </p>
    {% endblock %}

    {% block action %}
        <spacer size="16"></spacer>
        <button href="{{ url('review_comment', { id: comment.id }) }}">Accept</button>
        <button href="{{ url('review_comment', { id: comment.id, reject: true }) }}">Reject</button>
    {% endblock %}

The template overrides a few blocks to customize the message of the email and to add some links that allow the admin to accept or reject a comment. Any route argument that is not a valid route parameter is added as a query string item (the reject URL looks like ``/admin/comment/review/42?reject=true``).

The default ``NotificationEmail`` template uses `Inky`_ instead of HTML to design emails. It helps create responsive emails that are compatible with all popular email clients.

For maximum compatibility with email readers, the notification base layout inlines all stylesheets (via the CSS inliner package) by default.

These two features are part of optional Twig extensions that need to be installed:

.. code-block:: terminal

    $ symfony composer req "twig/cssinliner-extra:^3" "twig/inky-extra:^3"

Generating Absolute URLs in a Symfony Command
---------------------------------------------

.. index::
    single: Twig;Link
    single: Link

In emails, generate URLs with ``url()`` instead of ``path()`` as you need absolute ones (with scheme and host).

The email is sent from the message handler, in a console context. Generating absolute URLs in a Web context is easier as we know the scheme and domain of the current page. This is not the case in a console context.

Define the domain name and scheme to use explicitly:

.. code-block:: diff
    :caption: patch_file

    --- i/config/services.yaml
    +++ w/config/services.yaml
    @@ -7,6 +7,7 @@ parameters:
         photo_dir: "%kernel.project_dir%/public/uploads/photos"
         default_admin_email: admin@example.com
         admin_email: "%env(string:default:default_admin_email:ADMIN_EMAIL)%"
    +    default_base_url: 'http://127.0.0.1'

     services:
         # default configuration for services in *this* file

Then tell the router to use it as the default URI when generating URLs outside of an HTTP request:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/routing.yaml
    +++ w/config/packages/routing.yaml
    @@ -3,3 +3,3 @@ framework:
             # Configure how to generate URLs in non-HTTP contexts, such as CLI commands.
             # See https://symfony.com/doc/current/routing.html#generating-urls-in-commands
    -        default_uri: '%env(DEFAULT_URI)%'
    +        default_uri: '%env(default:default_base_url:SYMFONY_DEFAULT_ROUTE_URL)%'

The ``SYMFONY_DEFAULT_ROUTE_URL`` environment variable is automatically set locally when using the ``symfony`` CLI and determined based on the configuration on Upsun.

Wiring a Route to a Controller
------------------------------

The ``review_comment`` route does not exist yet, let's create an admin controller to handle it:

.. code-block:: php
    :caption: src/Controller/AdminController.php

    namespace App\Controller;

    use App\Entity\Comment;
    use App\Message\CommentMessage;
    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\DependencyInjection\Attribute\Target;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Messenger\MessageBusInterface;
    use Symfony\Component\Routing\Attribute\Route;
    use Symfony\Component\Workflow\WorkflowInterface;
    use Twig\Environment;

    class AdminController extends AbstractController
    {
        public function __construct(
            private Environment $twig,
            private EntityManagerInterface $entityManager,
            private MessageBusInterface $bus,
        ) {
        }

        #[Route('/admin/comment/review/{id}', name: 'review_comment')]
        public function reviewComment(Request $request, Comment $comment, #[Target('comment')] WorkflowInterface $workflow): Response
        {
            $accepted = !$request->query->get('reject');

            if ($workflow->can($comment, 'publish')) {
                $transition = $accepted ? 'publish' : 'reject';
            } elseif ($workflow->can($comment, 'publish_ham')) {
                $transition = $accepted ? 'publish_ham' : 'reject_ham';
            } else {
                return new Response('Comment already reviewed or not in the right state.');
            }

            $workflow->apply($comment, $transition);
            $this->entityManager->flush();

            if ($accepted) {
                $this->bus->dispatch(new CommentMessage($comment->getId()));
            }

            return new Response($this->twig->render('admin/review.html.twig', [
                'transition' => $transition,
                'comment' => $comment,
            ]));
        }
    }

The review comment URL starts with ``/admin/`` to protect it with the firewall defined in a previous step. The admin needs to be authenticated to access this resource.

Instead of creating a ``Response`` instance, we have used ``render()``, a shortcut method provided by the ``AbstractController`` controller base class.

.. index::
    single: Twig;extends
    single: Twig;block

When the review is done, a short template thanks the admin for their hard work:

.. code-block:: html+twig
    :caption: templates/admin/review.html.twig

    {% extends 'base.html.twig' %}

    {% block body %}
        <h2>Comment reviewed, thank you!</h2>

        <p>Applied transition: <strong>{{ transition }}</strong></p>
        <p>New state: <strong>{{ comment.state }}</strong></p>
    {% endblock %}

Using a Mail Catcher
--------------------

.. index::
    single: Docker;Mail Catcher

Instead of using a "real" SMTP server or a third-party provider to send emails, let's use a mail catcher. A mail catcher provides an SMTP server that does not deliver the emails, but makes them available through a Web interface instead. Fortunately, Symfony has already configured such a mail catcher automatically for us:

.. code-block:: yaml
    :caption: compose.override.yaml
    :class: ignore

    ###> symfony/mailer ###
    mailer:
        image: axllent/mailpit
        ports:
        - "1025"
        - "8025"
        environment:
        MP_SMTP_AUTH_ACCEPT_ANY: 1
        MP_SMTP_AUTH_ALLOW_INSECURE: 1
    ###< symfony/mailer ###

Accessing the Webmail
---------------------

.. index::
    single: Symfony CLI;open:local:webmail

You can open the webmail from a terminal:

.. code-block:: terminal
    :class: ignore

    $ symfony open:local:webmail

Or from the web debug toolbar:

.. figure:: screenshots/webmail-wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Submit a comment, you should receive an email in the webmail interface:

.. figure:: screenshots/webmail.png
    :alt: /
    :align: center
    :figclass: with-browser

Click on the email title on the interface and accept or reject the comment as you see fit:

.. figure:: screenshots/webmail-rejected.png
    :alt: /
    :align: center
    :figclass: with-browser

Check the logs with ``server:log`` if that does not work as expected.

Managing Long-Running Scripts
-----------------------------

Having long-running scripts comes with behaviors that you should be aware of. Unlike the PHP model used for HTTP where each request starts with a clean state, the message consumer is running continuously in the background. Each handling of a message inherits the current state, including the memory cache. To avoid any issues with Doctrine, its entity managers are automatically cleared after the handling of a message. You should check if your own services need to do the same or not.

Sending Emails Asynchronously
-----------------------------

The email sent in the message handler might take some time to be sent. It might even throw an exception. In case of an exception being thrown during the handling of a message, it will be retried. But instead of retrying to consume the comment message, it would be better to actually just retry sending the email.

We already know how to do that: send the email message on the bus.

A ``MailerInterface`` instance does the hard work: when a bus is defined, it dispatches the email messages on it instead of sending them. No changes are needed in your code.

The bus is already sending the email asynchronously as per the default Messenger configuration:

.. code-block:: yaml
    :caption: config/packages/messenger.yaml
    :emphasize-lines: 4
    :class: ignore

    framework:
        messenger:
            routing:
                Symfony\Component\Mailer\Messenger\SendEmailMessage: async
                Symfony\Component\Notifier\Message\ChatMessage: async
                Symfony\Component\Notifier\Message\SmsMessage: async

                # Route your messages to the transports
                App\Message\CommentMessage: async

Even if we are using the same transport for comment messages and email messages, it does not have to be the case. You could decide to use another transport to manage different message priorities for instance. Using different transports also gives you the opportunity to have different worker machines handling different kinds of messages. It is flexible and up to you.

Testing Emails
--------------

There are many ways to test emails.

You can write unit tests if you write a class per email (by extending ``Email`` or ``TemplatedEmail`` for instance).

The most common tests you will write though are functional tests that check that some actions trigger an email, and probably tests about the content of the emails if they are dynamic.

Symfony comes with assertions that ease such tests, here is a test example that demonstrates some possibilities:

.. code-block:: php
    :class: ignore

    public function testMailerAssertions(): void
    {
        $client = static::createClient();
        $client->request('GET', '/');

        $this->assertEmailCount(1);
        $event = $this->getMailerEvent(0);
        $this->assertEmailIsQueued($event);

        $email = $this->getMailerMessage(0);
        $this->assertEmailHeaderSame($email, 'To', 'fabien@example.com');
        $this->assertEmailTextBodyContains($email, 'Bar');
        $this->assertEmailAttachmentCount($email, 1);
    }

These assertions work when emails are sent synchronously or asynchronously.

Sending Emails on Upsun
-----------------------

.. index::
    single: Upsun;Emails
    single: Upsun;Mailer
    single: Upsun;SMTP
    single: Emails

There is no specific configuration for Upsun. All accounts come with a SendGrid account that is automatically used to send emails.

.. index::
    single: Symfony CLI;cloud:env:info

.. note::

    To be on the safe side, emails are *only* sent on the ``master`` branch by default. Enable SMTP explicitly on non-``master`` branches if you know what you are doing:

    .. code-block:: terminal

        $ symfony cloud:env:info enable_smtp on

.. sidebar:: Going Further

    * `SymfonyCasts Mailer tutorial`_;

    * The `Inky templating language docs`_;

    * The `Environment Variable Processors`_;

    * The `Symfony Framework Mailer documentation`_;

    * The `Upsun documentation about Emails`_.

.. _`Inky`: https://get.foundation/emails/docs/inky.html
.. _`SymfonyCasts Mailer tutorial`: https://symfonycasts.com/screencast/mailer
.. _`Inky templating language docs`: https://get.foundation/emails/docs/inky.html
.. _`Environment Variable Processors`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`Symfony Framework Mailer documentation`: https://symfony.com/doc/current/mailer.html
.. _`Upsun documentation about Emails`: https://symfony.com/doc/current/cloud/services/emails.html
