تغییر اندازه‌ی تصاویر
=========================================

در طراحی صفحه‌ی کنفرانس، اندازه تصاویر به ۲۰۰ در ۱۵۰ پیکسل محدود شده است. پس بهینه‌سازی و کاهش حجم تصاویر در صورتی که نسخه‌ی اصلی بارگذاری‌شده از حدود تعیین‌شده بیشتر باشد چه می‌شود؟

این یک کار فوق‌العاده است که می‌تواند به جریان‌کار کامنت اضافه شود. احتمالاً درست پس از اعتبارسنجی کامنت و قبل از انتشار آن.

بیایید یک وضعیت جدید ``ready`` و یک تحول ``optimize`` اضافه کنیم:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/workflow.yaml
    +++ w/config/packages/workflow.yaml
    @@ -16,6 +16,7 @@ framework:
                     - potential_spam
                     - spam
                     - rejected
    +                - ready
                     - published
                 transitions:
                     accept:
    @@ -29,13 +30,16 @@ framework:
                         to:   spam
                     publish:
                         from: potential_spam
    -                    to:   published
    +                    to:   ready
                     reject:
                         from: potential_spam
                         to:   rejected
                     publish_ham:
                         from: ham
    -                    to:   published
    +                    to:   ready
                     reject_ham:
                         from: ham
                         to:   rejected
    +                optimize:
    +                    from: ready
    +                    to:   published

.. index::
    single: Command;workflow:dump

ارائه‌ی بصری پیکربندی جریان‌کار جدید را تولید کنید تا تصدیق نمایید که همان چیزی است که می‌خواهید:

.. code-block:: terminal
    :class: ignore

    $ symfony console workflow:dump comment | dot -Tpng -o workflow.png

.. image:: images/workflow-final.png
    :align: center

بهینه‌سازی تصاویر با Imagine
-----------------------------------------------

.. index::
    single: Imagine

بهینه‌سازی تصاویر به لطف `GD`_ (بررسی کنید که افزونه‌ی GD در نصب محلی PHP فعال باشد) و `Imagine`_ انجام می‌گردد:

.. code-block:: terminal

    $ symfony composer req "imagine/imagine:^1.5"

تغییر اندازه‌ی تصاویر می‌تواند به کمک کلاس سرویس زیر انجام شود:

.. code-block:: php
    :caption: src/ImageOptimizer.php

    namespace App;

    use Imagine\Gd\Imagine;
    use Imagine\Image\Box;

    class ImageOptimizer
    {
        private const MAX_WIDTH = 200;
        private const MAX_HEIGHT = 150;

        private readonly Imagine $imagine;

        public function __construct()
        {
            $this->imagine = new Imagine();
        }

        public function resize(string $filename): void
        {
            [$iwidth, $iheight] = getimagesize($filename);
            $ratio = $iwidth / $iheight;
            $width = self::MAX_WIDTH;
            $height = self::MAX_HEIGHT;
            if ($width / $height > $ratio) {
                $width = $height * $ratio;
            } else {
                $height = $width / $ratio;
            }

            $photo = $this->imagine->open($filename);
            $photo->resize(new Box($width, $height))->save($filename);
        }
    }

ما پس از بهینه‌سازی تصاویر، فایل جدید را در همان مکان تصویر اصلی ذخیره می‌کنیم. البته شما ممکن است بخواهید تصویر اصلی را نگه‌دارید.

افزودن یک گام جدید در جریان‌کار
----------------------------------------------------------

جریان‌کار را اصلاح کنید تا به یک وضعیت جدید رسیدگی کند:

.. code-block:: diff
    :caption: patch_file

    --- i/src/MessageHandler/CommentMessageHandler.php
    +++ w/src/MessageHandler/CommentMessageHandler.php
    @@ -2,6 +2,7 @@

     namespace App\MessageHandler;

    +use App\ImageOptimizer;
     use App\Message\CommentMessage;
     use App\Repository\CommentRepository;
     use App\SpamChecker;
    @@ -25,6 +26,8 @@ class CommentMessageHandler
             private WorkflowInterface $commentStateMachine,
             private MailerInterface $mailer,
             #[Autowire('%admin_email%')] private string $adminEmail,
    +        private ImageOptimizer $imageOptimizer,
    +        #[Autowire('%photo_dir%')] private string $photoDir,
             private ?LoggerInterface $logger = null,
         ) {
         }
    @@ -54,6 +57,12 @@ class CommentMessageHandler
                     ->to($this->adminEmail)
                     ->context(['comment' => $comment])
                 );
    +        } elseif ($this->commentStateMachine->can($comment, 'optimize')) {
    +            if ($comment->getPhotoFilename()) {
    +                $this->imageOptimizer->resize($this->photoDir.'/'.$comment->getPhotoFilename());
    +            }
    +            $this->commentStateMachine->apply($comment, 'optimize');
    +            $this->entityManager->flush();
             } elseif ($this->logger) {
                 $this->logger->debug('Dropping comment message', ['comment' => $comment->getId(), 'state' => $comment->getState()]);
             }

توجه کنید که ``$photoDir`` به صورت خودکار تزریق شده است، چرا که ما در گامی پیشین یک *پارامتر (parameter)* کانتینر بر روی نام این متغیر تعریف کردیم:

.. code-block:: yaml
    :caption: config/services.yaml
    :class: ignore

    parameters:
        photo_dir: "%kernel.project_dir%/public/uploads/photos"

ذخیره‌ی داده‌های بارگذاری‌شده در محیط عمل‌آوری
-------------------------------------------------------------------------------------------

.. index::
    single: Upsun;File Service

ما در حال حاضر یک پوشه‌ی مخصوص قابل خواندن و نوشتن برای فایل‌های باگذاری‌شده در ``.upsun/config.yaml`` تعریف کرده‌ایم. اما mount آن نسبت به کانتینر اپلیکیشن محلی است. اگر می‌خواهیم که کانتینر وب و کارگر مصرف‌کننده‌ی پیغام، قادر به دسترسی به همین mount مشترک باشند، لازم است که یک *سرویس فایل* ایجاد کنیم:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -15,6 +15,9 @@ services:
                     type: string
                     path: config.vcl

    +    files:
    +        type: network-storage:2.0
    +
     applications:

از آن به عنوان پوشه‌ی بارگذاری تصاویر استفاده کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -54,7 +54,7 @@ applications:
             mounts:
                 "/var/cache": { source: instance, source_path: var/cache }
                 "/var/share": { source: storage, source_path: var/share }
    -            "/public/uploads": { source: storage, source_path: uploads }
    +            "/public/uploads": { source: service, service: files, source_path: uploads }


             relationships:

انجام کارهای اخیر باید برای کارکردن قابلیت‌ها در محیط عمل‌آوری کافی باشد.

.. _`GD`: https://libgd.github.io/
.. _`Imagine`: https://github.com/avalanche123/Imagine
