زمان‌بندی وظایف
==========================

.. index::
    single: Cron

برخی وظایف نگهداری باید بر اساس یک زمان‌بندی اجرا شوند. برخلاف کارگرها (workers) که به صورت مستمر اجرا می‌شوند، وظایف زمان‌بندی‌شده به صورت دوره‌ای و برای مدت کوتاهی اجرا می‌گردند.

پاک‌سازی کامنت‌ها
------------------------------

کامنت‌هایی که به عنوان هرز علامت‌گذاری شده یا توسط مدیر رد شده‌اند، در پایگاه‌داده نگه داشته می‌شوند، زیرا مدیر ممکن است بخواهد آن‌ها را برای مدتی کوتاه بررسی کند. اما احتمالاً باید پس از مدتی حذف شوند. نگه‌داشتن آن‌ها به مدت یک هفته پس از ایجادشان، احتمالاً کافی است.

چند متد کمکی در مخزن کامنت ایجاد کنید تا کامنت‌های ردشده را پیدا کرده، آن‌ها را بشمارد و حذف کند:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Repository/CommentRepository.php
    +++ w/src/Repository/CommentRepository.php
    @@ -5,7 +5,9 @@ namespace App\Repository;
     use App\Entity\Comment;
     use App\Entity\Conference;
     use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    +use Doctrine\Common\Collections\ArrayCollection;
     use Doctrine\Persistence\ManagerRegistry;
    +use Doctrine\ORM\QueryBuilder;
     use Doctrine\ORM\Tools\Pagination\Paginator;

     /**
    @@ -13,6 +15,8 @@ use Doctrine\ORM\Tools\Pagination\Paginator;
      */
     class CommentRepository extends ServiceEntityRepository
     {
    +    private const DAYS_BEFORE_REJECTED_REMOVAL = 7;
    +
         public const COMMENTS_PER_PAGE = 2;

         public function __construct(ManagerRegistry $registry)
    @@ -20,6 +24,27 @@ class CommentRepository extends ServiceEntityRepository
             parent::__construct($registry, Comment::class);
         }

    +    public function countOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->select('COUNT(c.id)')->getQuery()->getSingleScalarResult();
    +    }
    +
    +    public function deleteOldRejected(): int
    +    {
    +        return $this->getOldRejectedQueryBuilder()->delete()->getQuery()->execute();
    +    }
    +
    +    private function getOldRejectedQueryBuilder(): QueryBuilder
    +    {
    +        return $this->createQueryBuilder('c')
    +            ->andWhere('c.state = :state_rejected or c.state = :state_spam')
    +            ->andWhere('c.createdAt < :date')
    +            ->setParameter('state_rejected', 'rejected')
    +            ->setParameter('state_spam', 'spam')
    +            ->setParameter('date', new \DateTimeImmutable(-self::DAYS_BEFORE_REJECTED_REMOVAL.' days'))
    +        ;
    +    }
    +
         public function getCommentPaginator(Conference $conference, int $offset): Paginator
         {
             $query = $this->createQueryBuilder('c')

.. tip::

    برای پرس‌وجوهای پیچیده‌تر، گاهی اوقات نگاه‌کردن به دستورات SQL تولیدشده مفید است (آن‌ها را می‌توان در لاگ‌ها و در نمایه‌ساز برای درخواست‌های وب یافت).

استفاده از ثابت‌های کلاس، پارامترهای کانتینر و متغیرهای محیط
------------------------------------------------------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

۷ روز؟ می‌توانستیم عدد دیگری انتخاب کنیم، شاید ۱۰ یا ۲۰. این عدد ممکن است در طول زمان تغییر کند. ما تصمیم گرفتیم آن را به عنوان یک ثابت روی کلاس ذخیره کنیم، اما می‌توانستیم آن را به عنوان یک پارامتر در کانتینر ذخیره کنیم، یا حتی می‌توانستیم آن را به عنوان یک متغیر محیط تعریف کنیم.

در اینجا چند قاعده‌ی سرانگشتی برای تصمیم‌گیری درباره‌ی اینکه از کدام انتزاع استفاده کنیم آمده است:

* اگر مقدار حساس است (رمزهای عبور، توکن‌های API و ...)، از *انبار رمز (secret storage)* سیمفونی یا یک Vault استفاده کنید؛

* اگر مقدار پویاست و باید بتوانید آن را *بدون* استقرار مجدد تغییر دهید، از یک *متغیر محیط* استفاده کنید؛

* اگر مقدار می‌تواند بین محیط‌ها متفاوت باشد، از یک *پارامتر کانتینر* استفاده کنید؛

* برای هر چیز دیگر، مقدار را در کد ذخیره کنید، مانند یک *ثابت کلاس*.

ایجاد یک فرمان CLI
--------------------------------

حذف کامنت‌های قدیمی، وظیفه‌ی مناسبی برای یک cron job است. باید به صورت منظم انجام شود و کمی تأخیر تأثیر عمده‌ای ندارد.

با ایجاد یک فایل ``src/Command/CommentCleanupCommand.php``، یک فرمان CLI با نام ``app:comment:cleanup`` ایجاد کنید:

.. code-block:: php
    :caption: src/Command/CommentCleanupCommand.php

    namespace App\Command;

    use App\Repository\CommentRepository;
    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Attribute\Option;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Style\SymfonyStyle;

    #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    class CommentCleanupCommand
    {
        public function __invoke(
            SymfonyStyle $io,
            CommentRepository $commentRepository,
            #[Option(description: 'Dry run')]
            bool $dryRun = false,
        ): int {
            if ($dryRun) {
                $io->note('Dry mode enabled');

                $count = $commentRepository->countOldRejected();
            } else {
                $count = $commentRepository->deleteOldRejected();
            }

            $io->success(sprintf('Deleted "%d" old rejected/spam comments.', $count));

            return Command::SUCCESS;
        }
    }

تمام فرمان‌های اپلیکیشن در کنار فرمان‌های توکار سیمفونی ثبت می‌شوند و همگی از طریق ``symfony console`` قابل دسترسی هستند. از آنجایی که تعداد فرمان‌های موجود می‌تواند زیاد باشد، باید آن‌ها را در فضای نام قرار دهید. طبق قرارداد، فرمان‌های اپلیکیشن باید در فضای نام ``app`` ذخیره شوند. هر تعداد زیرفضای‌نام را با جداکردن آن‌ها به وسیله‌ی یک علامت دونقطه (``:``) اضافه کنید.

یک فرمان *آرگمان‌ها* و *گزینه‌های* خود را با attributeهای ``#[Argument]`` و ``#[Option]`` روی پارامترهای ``__invoke()`` اعلام می‌کند (پارامتر ``$dryRun`` به گزینه‌ی ``--dry-run`` تبدیل می‌شود). سیمفونی سایر پارامترها را بر اساس نوعشان تزریق می‌کند: ``SymfonyStyle`` برای نوشتن خروجی خوش‌قالب به کنسول، و هر سرویسی مانند مخزن کامنت، به همان روشی که برای آرگمان‌های کنترلر انجام می‌دهد.

با اجرای این فرمان، پایگاه‌داده را پاک‌سازی کنید:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

زمان‌بندی فرمان
------------------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

اجرای دستی فرمان کار می‌کند، اما باید هر شب اجرا شود. کامپوننت Scheduler سیمفونی، پیغام‌هایی را بر اساس یک زمان‌بندی تولید می‌کند؛ سپس این پیغام‌ها مانند هر پیغام پیغام‌رسان دیگری، توسط یک کارگر مصرف می‌شوند.

کامپوننت Scheduler را به همراه کتابخانه‌ای که عبارات cron را تجزیه می‌کند، اضافه کنید:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

فرمان را با attribute‌ی ``#[AsCronTask]`` زمان‌بندی کنید:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/CommentCleanupCommand.php
    +++ w/src/Command/CommentCleanupCommand.php
    @@ -7,8 +7,10 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Attribute\Option;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Style\SymfonyStyle;
    +use Symfony\Component\Scheduler\Attribute\AsCronTask;

     #[AsCommand('app:comment:cleanup', 'Deletes rejected and spam comments from the database')]
    +#[AsCronTask('50 23 * * *')]
     class CommentCleanupCommand
     {
         public function __invoke(

این attribute فرمان را با یک عبارت cron روی *زمان‌بندیِ* پیش‌فرض ثبت می‌کند: هر شب ساعت ۱۱:۵۰ بعدازظهر (به وقت UTC). آن را بررسی کنید:

.. code-block:: terminal

    $ symfony console debug:scheduler

یک زمان‌بندی به صورت یک صف معمولی پیغام‌رسان که هم‌نام آن است عرضه می‌شود؛ آن را مانند هر صف دیگری مصرف کنید:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

استقرار زمان‌بندی
------------------------------

.. index::
    single: Upsun;Workers

در Upsun، کارگر تنها صف ``async`` را مصرف می‌کند. آن را وادارید تا زمان‌بندی را نیز مصرف کند:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -87,4 +87,4 @@ applications:
             messenger:
                 commands:
                     # Consume "async" messages (as configured in the routing section of config/packages/messenger.yaml)
    -                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async
    +                    start: symfony console --time-limit=3600 --memory-limit=64M messenger:consume async scheduler_default

همین کافی است: بدون crontab، بدون فرایند اضافه؛ زمان‌بندی در کد PHP، در کنار وظیفه‌ای که آن را راه‌اندازی می‌کند، زندگی می‌کند و مانند باقی اپلیکیشن مستقر و نسخه‌بندی می‌شود.

cronهای سیستمی چطور؟
------------------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun از cron jobهای سطح سیستم‌عامل نیز پشتیبانی می‌کند که در ``.upsun/config.yaml`` در کنار کانتینر وب و کارگرها توصیف می‌شوند؛ پیکربندی پیش‌فرض از پیش یکی را تعریف می‌کند که نشست‌های منقضی‌شده‌ی PHP را پاک می‌کند. cronهای سیستمی برای وظایفی که در PHP پیاده‌سازی نشده‌اند، گزینه‌ی مناسبی هستند.

ابزار ``croncape`` که توسط cron پیش‌فرض استفاده می‌شود، اجرای فرمان را پایش می‌کند و در صورتی که فرمان هر کد خروجی متفاوت از ``0`` بازگرداند، یک رایانامه به آدرس‌های تعریف‌شده در متغیر محیط ``MAILTO`` می‌فرستد:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

توجه کنید که cronها روی تمام شاخه‌های Upsun راه‌اندازی می‌شوند. اگر نمی‌خواهید برخی را روی محیط‌های غیرعمل‌آوری اجرا کنید، متغیر محیط ``$PLATFORM_ENVIRONMENT_TYPE`` را بررسی کنید:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: بیشتر بدانید

    * `مستندات کامپوننت Scheduler`_؛

    * `نحو Cron/crontab`_؛

    * `مخزن Croncape`_؛

    * `فرمان‌های کنسول سیمفونی`_؛

    * `برگه‌تقلب کنسول سیمفونی`_.

.. _`مستندات کامپوننت Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`نحو Cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`مخزن Croncape`: https://github.com/symfonycorp/croncape
.. _`فرمان‌های کنسول سیمفونی`: https://symfony.com/doc/current/console.html
.. _`برگه‌تقلب کنسول سیمفونی`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
