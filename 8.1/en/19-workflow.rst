Making Decisions with a Workflow
================================

.. index::
    single: Components;Workflow
    single: Workflow

Having a state for a model is quite common. The comment state is only determined by the spam checker. What if we add more decision factors?

We might want to let the website admin moderate all comments after the spam checker. The process would be something along the lines of:

* Start with a ``submitted`` state when a comment is submitted by a user;

* Let the spam checker analyze the comment and switch the state to either ``potential_spam``, ``ham``, or ``rejected``;

* If not rejected, wait for the website admin to decide if the comment is good enough by switching the state to ``published`` or ``rejected``.

Implementing this logic is not too complex, but you can imagine that adding more rules would greatly increase the complexity. Instead of coding the logic ourselves, we can use the Symfony Workflow Component:

.. code-block:: terminal

    $ symfony composer req workflow

Describing Workflows
--------------------

The comment workflow can be described in the ``config/packages/workflow.yaml`` file:

.. code-block:: yaml
    :caption: config/packages/workflow.yaml
    :emphasize-lines: 3,4,9,11

    framework:
        workflows:
            comment:
                type: state_machine
                audit_trail:
                    enabled: "%kernel.debug%"
                marking_store:
                    type: 'method'
                    property: 'state'
                supports:
                    - App\Entity\Comment
                initial_marking: submitted
                places:
                    - submitted
                    - ham
                    - potential_spam
                    - spam
                    - rejected
                    - published
                transitions:
                    accept:
                        from: submitted
                        to:   ham
                    might_be_spam:
                        from: submitted
                        to:   potential_spam
                    reject_spam:
                        from: submitted
                        to:   spam
                    publish:
                        from: potential_spam
                        to:   published
                    reject:
                        from: potential_spam
                        to:   rejected
                    publish_ham:
                        from: ham
                        to:   published
                    reject_ham:
                        from: ham
                        to:   rejected

.. index::
    single: Command;workflow:dump

To validate the workflow, generate a visual representation in the Mermaid format:

.. code-block:: terminal

    $ symfony console workflow:dump comment --dump-format=mermaid

Paste the output in the `Mermaid Live Editor`_ to render it; GitHub and GitLab also render Mermaid diagrams natively in Markdown files:

.. image:: images/workflow.png
    :align: center

Using a Workflow
----------------

Replace the current logic in the message handler with the workflow:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -6,7 +6,11 @@ use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
    +use Psr\Log\LoggerInterface;
    +use Symfony\Component\DependencyInjection\Attribute\Target;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
    +use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
     class CommentMessageHandler
    @@ -15,6 +19,9 @@ class CommentMessageHandler
             private EntityManagerInterface $entityManager,
             private SpamChecker $spamChecker,
             private CommentRepository $commentRepository,
    +        private MessageBusInterface $bus,
    +        #[Target('comment')] private WorkflowInterface $workflow,
    +        private ?LoggerInterface $logger = null,
         ) {
         }

    @@ -25,12 +32,18 @@ class CommentMessageHandler
                 return;
             }

    -        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
    -            $comment->setState('spam');
    -        } else {
    -            $comment->setState('published');
    +        if ($this->workflow->can($comment, 'accept')) {
    +            $score = $this->spamChecker->getSpamScore($comment, $message->getContext());
    +            $transition = match ($score) {
    +                2 => 'reject_spam',
    +                1 => 'might_be_spam',
    +                default => 'accept',
    +            };
    +            $this->workflow->apply($comment, $transition);
    +            $this->entityManager->flush();
    +            $this->bus->dispatch($message);
    +        } elseif ($this->logger) {
    +            $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }
    -
    -        $this->entityManager->flush();
         }
     }

The new logic reads as follows:

* If the ``accept`` transition is available for the comment in the message, check for spam;

* Depending on the outcome, choose the right transition to apply;

* Call ``apply()`` to update the Comment via a call to the ``setState()`` method;

* Call ``flush()`` to commit the changes to the database;

* Re-dispatch the message to allow the workflow to transition again.

As we haven't implemented the admin validation, the next time the message is consumed, the "Dropping comment message" will be logged.

Let's implement an auto-validation until the next chapter:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -42,6 +42,9 @@ class CommentMessageHandler
                 $this->workflow->apply($comment, $transition);
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
    +        } elseif ($this->workflow->can($comment, 'publish') || $this->workflow->can($comment, 'publish_ham')) {
    +            $this->workflow->apply($comment, $this->workflow->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

Run ``symfony server:log`` and add a comment in the frontend to see all transitions happening one after the other.

Finding Services from the Dependency Injection Container
--------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

When using dependency injection, we get services from the dependency injection container by type hinting an interface or sometimes a concrete implementation class name. But when an interface has several implementations, Symfony cannot guess which one you need. We need a way to be explicit.

We have just come across such an example with the injection of a ``WorkflowInterface`` in the previous section.

That is why we added the ``#[Target('comment')]`` attribute on the ``WorkflowInterface`` argument in the constructor: it tells Symfony to inject the ``comment`` workflow defined in the configuration (whose type is ``state_machine``).

If you don't remember the workflow name, use the ``debug:container`` command. Search for all services containing "workflow":

.. code-block:: terminal
    :emphasize-lines: 12
    :class: ignore

    $ symfony console debug:container workflow

     Select one of the following services to display its information:
      [0] console.command.workflow_dump
      [1] workflow.abstract
      [2] workflow.marking_store.method
      [3] workflow.registry
      [4] workflow.security.expression_language
      [5] workflow.twig_extension
      [6] monolog.logger.workflow
      [7] Symfony\Component\Workflow\Registry
      [8] Symfony\Component\Workflow\WorkflowInterface $commentStateMachine
      [9] Psr\Log\LoggerInterface $workflowLogger
     >

Notice choice ``8``, ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine``: it is the autowiring alias Symfony generated for the ``comment`` state machine. Pass the workflow name to ``#[Target]`` to inject it.

.. note::

    We could have used the ``debug:autowiring`` command as seen in a previous chapter:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: Going Further

    * `Workflows and State Machines`_ and when to choose each one;

    * The `Symfony Workflow docs`_.

.. _`Mermaid Live Editor`: https://mermaid.live/
.. _`Workflows and State Machines`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`Symfony Workflow docs`: https://symfony.com/doc/current/workflow.html
