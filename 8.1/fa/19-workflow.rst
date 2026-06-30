تصمیم‌گیری با یک جریان‌کار
===================================================

.. index::
    single: Components;Workflow
    single: Workflow

داشتن یک وضعیت برای یک مدل، کاملاً عادی است. وضعیت یک کامنت، تنها توسط بررسی‌کننده‌ی محتوای هرز تعیین شده است. اگر ما فاکتورهای تصمیم‌گیری بیشتری اضافه کنیم، چه اتفاقی می‌افتد؟

ما ممکن است بخواهیم که پس از بررسی‌کننده‌ی داده‌های هرز، مدیر سایت نیز کامنت‌ها را تعدیل کند. فرآیند رویه‌ای مشابه زیر خواهد داشت:

* هنگامی که یک کامنت ارسال شد، با وضعیت ``submitted`` شروع کن؛

* بگذار که بررسی‌کننده‌ی داده‌های هرز، کامنت را تحلیل کند و وضعیت را به یکی از وضعیت‌های ``potential_spam``، ``ham`` یا ``rejected`` تغییر دهد.

* اگر کامنت رد نشد، صبر کن تا مدیر وب‌سایت با تغییر وضعیت به ``published`` یا ``rejected``، تصمیم بگیرد که آیا کامنت به اندازه‌ی کافی خوب است یا خیر.

پیاده‌سازی این منطق خیلی پیچیده نیست، اما شما می‌توانید تصور کنید که اضافه‌شدن قواعد بیشتر، پیچیدگی را به میزان زیادی افزایش خواهد داد. ما می‌توانیم به جای اینکه خودمان این منطق را کدنویسی کنیم، از کامپوننت جریان‌کار سیمفونی اسفاده کنیم:

.. code-block:: terminal

    $ symfony composer req workflow

توصیف جریان‌کارها
----------------------------------

جریان‌کار کامنت می‌تواند در فایل ``config/packages/workflow.yaml`` توصیف گردد:

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

برای اعتبارسنجی جریان‌کار، یک ارائه‌ی تصویری در قالب Mermaid تولید کنید:

.. code-block:: terminal

    $ symfony console workflow:dump comment --dump-format=mermaid

خروجی را در `ویرایشگر زنده‌ی Mermaid`_ بچسبانید تا render شود؛ GitHub و GitLab نیز نمودارهای Mermaid را به‌صورت بومی در فایل‌های Markdown رندر می‌کنند:

.. image:: images/workflow.png
    :align: center

استفاده از یک جریان‌کار
--------------------------------------------

منطق موجود در رسیدگی‌کننده‌ی پیغام را با جریان‌کار جایگزین کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -6,7 +6,10 @@ use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
    +use Psr\Log\LoggerInterface;
     use Symfony\Component\Messenger\Attribute\AsMessageHandler;
    +use Symfony\Component\Messenger\MessageBusInterface;
    +use Symfony\Component\Workflow\WorkflowInterface;

     #[AsMessageHandler]
     class CommentMessageHandler
    @@ -15,6 +18,9 @@ class CommentMessageHandler
             private EntityManagerInterface $entityManager,
             private SpamChecker $spamChecker,
             private CommentRepository $commentRepository,
    +        private MessageBusInterface $bus,
    +        private WorkflowInterface $commentStateMachine,
    +        private ?LoggerInterface $logger = null,
         ) {
         }

    @@ -25,12 +31,18 @@ class CommentMessageHandler
                 return;
             }

    -        if (2 === $this->spamChecker->getSpamScore($comment, $message->getContext())) {
    -            $comment->setState('spam');
    -        } else {
    -            $comment->setState('published');
    +        if ($this->commentStateMachine->can($comment, 'accept')) {
    +            $score = $this->spamChecker->getSpamScore($comment, $message->getContext());
    +            $transition = match ($score) {
    +                2 => 'reject_spam',
    +                1 => 'might_be_spam',
    +                default => 'accept',
    +            };
    +            $this->commentStateMachine->apply($comment, $transition);
    +            $this->entityManager->flush();
    +            $this->bus->dispatch($message);
    +        } elseif ($this->logger) {
    +            $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }
    -
    -        $this->entityManager->flush();
         }
     }

منطق جدید به این صورت خوانده می‌شود:

* اگر برای کامنت موجود در پیغام، تحولِ ``accept`` در دسترس است، هرز‌بودن محتوای آن را بررسی کن؛

* بسته به نتیجه‌ی این بررسی، تحول (transition) صحیح را برای اعمال‌کردن انتخاب کن؛

* ``apply()`` را فراخوانی کن تا از طریق فراخوانی متد ``setState()``، شیءِ Comment را به‌روزرسانی کند؛

* ``flush()`` را فرخوانی کن تا تغییرات در پایگاه‌داده اعمال شود؛

* پیغام را بازاعزام (Re-dispatch) کن تا اجازه داده شود که جریان‌کار دوباره به کار بیافتد؛

از آنجایی که هنوز اعتبارسنجی مدیر را پیاده‌سازی نکرده‌ایم، دفعه‌ی بعدی که پیغام مصرف شود، «Dropping comment message» لاگ خواهد شد.

بیایید تا قبل از یک پیاده‌سازی واقعی در فصل آینده، فعلاً یک اعتبارسنجی خودکار را پیاده‌سازی کنیم:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -41,6 +41,9 @@ class CommentMessageHandler
                 $this->commentStateMachine->apply($comment, $transition);
                 $this->entityManager->flush();
                 $this->bus->dispatch($message);
    +        } elseif ($this->commentStateMachine->can($comment, 'publish') || $this->commentStateMachine->can($comment, 'publish_ham')) {
    +            $this->commentStateMachine->apply($comment, $this->commentStateMachine->can($comment, 'publish') ? 'publish' : 'publish_ham');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

``symfony server:log`` را اجرا کنید و یک کامنت در جلوی صحنه ایجاد کنید تا تمام وقوع تحولات را یکی پس از دیگری ببینید.

یافتن سرویس‌ها از کانتینر تزریق وابستگی
------------------------------------------------------------------

.. index::
    single: Command;debug:container
    single: Container;Debug
    single: Debug;Container

هنگام استفاده از تزریق وابستگی، سرویس‌ها را با type hint کردن یک interface یا گاهی نام یک کلاس پیاده‌سازی مشخص، از کانتینر تزریق وابستگی می‌گیریم. اما وقتی یک interface چند پیاده‌سازی داشته باشد، سیمفونی نمی‌تواند حدس بزند که شما به کدام‌یک نیاز دارید. ما به راهی برای صریح‌بودن نیاز داریم.

ما همین الان در بخش قبلی با چنین نمونه‌ای، یعنی تزریق یک ``WorkflowInterface``، روبرو شدیم.

از آنجایی که ما هر نمونه‌ای از interface عمومیِ ``WorkflowInterface`` را در سازنده تزریق می‌کنیم، سیمفونی چگونه می‌تواند حدس بزند که کدام پیاده‌سازی جریان‌کار را باید استفاده کند؟ سیمفونی از یک قرارداد مبتنی بر نام آرگمان استفاده می‌کند: ``$commentStateMachine`` به جریان‌کار ``comment`` در پیکربندی اشاره می‌کند (که نوع آن ``state_machine`` است). هر نام آرگمان دیگری را امتحان کنید و شکست خواهد خورد.

اگر این قرارداد را به خاطر نمی‌آورید، از فرمان ``debug:container`` استفاده کنید. تمام سرویس‌های حاوی «workflow» را جستجو کنید:

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

به گزینه‌ی ``8``، یعنی ``Symfony\Component\Workflow\WorkflowInterface $commentStateMachine`` توجه کنید که به شما می‌گوید استفاده از ``$commentStateMachine`` به‌عنوان نام آرگمان معنای ویژه‌ای دارد.

.. note::

    می‌توانستیم از فرمان ``debug:autowiring`` که در فصلی پیشین دیدیم استفاده کنیم:

    .. code-block:: terminal

        $ symfony console debug:autowiring workflow

.. sidebar:: بیشتر بدانید

    * `جریان‌کارها و ماشین‌های حالت (State Machines)`_ و اینکه هر کدام را باید در چه شرایطی انتخاب کرد؛

    * `مستندات جریان‌کار سیمفونی`_.

.. _`ویرایشگر زنده‌ی Mermaid`: https://mermaid.live/
.. _`جریان‌کارها و ماشین‌های حالت (State Machines)`: https://symfony.com/doc/current/workflow/workflow-and-state-machine.html
.. _`مستندات جریان‌کار سیمفونی`: https://symfony.com/doc/current/workflow.html
