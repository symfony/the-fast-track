جدولة المهام
========================

.. index::
    single: Cron

يجب تشغيل بعض مهام الصيانة وفق جدول زمني. على عكس العمال (workers) الذين يعملون باستمرار، تعمل المهام المجدولة بشكل دوري ولفترة قصيرة من الوقت.

تنظيف التعليقات
----------------------

تُحفظ التعليقات المُعلَّمة كبريد عشوائي أو المرفوضة من قِبل المسؤول في قاعدة البيانات لأن المسؤول قد يرغب في فحصها لفترة وجيزة. لكن من المحتمل أن تُزال بعد مرور بعض الوقت. الاحتفاظ بها لمدة أسبوع بعد إنشائها ربما يكون كافيًا.

أنشئ بعض الطرق المساعدة في مستودع التعليقات (comment repository) للعثور على التعليقات المرفوضة وعدّها وحذفها:

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

    بالنسبة للاستعلامات الأكثر تعقيدًا، من المفيد أحيانًا إلقاء نظرة على عبارات SQL المُولَّدة (يمكن العثور عليها في السجلات وفي ملف التعريف (profiler) لطلبات الويب).

استخدام ثوابت الفئة ومُعاملات الحاوية ومتغيرات البيئة
-----------------------------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 أيام؟ كان بإمكاننا اختيار رقم آخر، ربما 10 أو 20. قد يتطور هذا الرقم بمرور الوقت. لقد قررنا تخزينه كثابت (constant) على الفئة، لكن كان بإمكاننا تخزينه كمُعامل في الحاوية، أو حتى تعريفه كمتغير بيئة.

إليك بعض القواعد الإرشادية لتقرير أي تجريد تستخدم:

* إذا كانت القيمة حساسة (كلمات مرور، رموز API، ...)، استخدم *تخزين الأسرار* في Symfony أو خزنة (Vault)؛

* إذا كانت القيمة ديناميكية ويجب أن تتمكن من تغييرها *دون* إعادة النشر، استخدم *متغير بيئة*؛

* إذا كانت القيمة قد تختلف بين البيئات، استخدم *مُعامل حاوية*؛

* لكل ما عدا ذلك، خزّن القيمة في الكود، مثل *ثابت فئة*.

إنشاء أمر CLI
----------------------

إزالة التعليقات القديمة مهمة مثالية لمهمة cron. يجب أن تتم بشكل منتظم، والتأخير البسيط ليس له أي تأثير كبير.

أنشئ أمر CLI باسم ``app:comment:cleanup`` عن طريق إنشاء ملف ``src/Command/CommentCleanupCommand.php``:

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

تُسجَّل جميع أوامر التطبيق جنبًا إلى جنب مع الأوامر المدمجة في Symfony، وجميعها متاحة عبر ``symfony console``. وبما أن عدد الأوامر المتاحة قد يكون كبيرًا، يجب أن تضعها في مساحة أسماء (namespace). حسب الاتفاق، يجب تخزين أوامر التطبيق تحت مساحة الأسماء ``app``. أضف أي عدد من مساحات الأسماء الفرعية بفصلها بنقطتين رأسيتين (``:``).

يُعلن الأمر عن *وسائطه* (arguments) و*خياراته* (options) عبر السمتين ``#[Argument]`` و ``#[Option]`` على مُعاملات ``__invoke()`` (يصبح المُعامل ``$dryRun`` هو الخيار ``--dry-run``). يحقن Symfony المُعاملات الأخرى بناءً على نوعها: ``SymfonyStyle`` لكتابة مخرجات مُنسَّقة بشكل جميل إلى وحدة التحكم، وأي خدمة، مثل مستودع التعليقات، بنفس الطريقة التي يفعلها مع وسائط وحدة التحكم.

نظّف قاعدة البيانات بتشغيل الأمر:

.. code-block:: terminal

    $ symfony console app:comment:cleanup

جدولة الأمر
----------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

تشغيل الأمر يدويًا يعمل، لكن يجب أن يعمل كل ليلة. يُولّد مكون Symfony Scheduler رسائل وفق جدول زمني؛ ثم يستهلكها عامل (worker)، مثل أي رسائل Messenger أخرى.

أضف مكون Scheduler، مع المكتبة التي تحلل تعبيرات cron:

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

جدول الأمر بالسمة ``#[AsCronTask]``:

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

تُسجّل السمة الأمرَ على *الجدول* (schedule) الافتراضي بتعبير cron: كل ليلة في الساعة 11:50 مساءً (بتوقيت UTC). تحقق منه:

.. code-block:: terminal

    $ symfony console debug:scheduler

يُكشَف الجدول كناقل (transport) Messenger عادي يحمل اسمه؛ استهلكه مثل أي ناقل آخر:

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

نشر الجدول
----------------------

.. index::
    single: Upsun;Workers

على Upsun، يستهلك العامل ناقل ``async`` فقط. اجعله يستهلك الجدول أيضًا:

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

هذا كل ما يلزم: لا crontab، ولا عملية إضافية؛ يعيش الجدول في كود PHP، بجوار المهمة التي يُطلقها، ويُنشَر ويُدار بالإصدارات مثل بقية التطبيق.

ماذا عن مهام cron على مستوى النظام؟
----------------------------------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

يدعم Upsun أيضًا مهام cron على مستوى نظام التشغيل، الموصوفة في ``.upsun/config.yaml`` جنبًا إلى جنب مع حاوية الويب والعمال؛ الإعداد الافتراضي يُعرّف بالفعل واحدة تنظف جلسات PHP منتهية الصلاحية. مهام cron على مستوى النظام مناسبة للمهام التي لا تُنفَّذ بلغة PHP.

تراقب أداة ``croncape`` المستخدمة في مهمة cron الافتراضية تنفيذ الأمر وترسل بريدًا إلكترونيًا إلى العناوين المُعرَّفة في متغير البيئة ``MAILTO`` إذا أرجع الأمر أي رمز خروج مختلف عن ``0``:

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

لاحظ أن مهام cron تُعَدّ على جميع فروع Upsun. إذا كنت لا تريد تشغيل بعضها على البيئات غير الإنتاجية، تحقق من متغير البيئة ``$PLATFORM_ENVIRONMENT_TYPE``:

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: الذهاب أبعد من ذلك

    * `مستندات مكون Scheduler`_؛

    * `صياغة Cron/crontab`_؛

    * `مستودع Croncape`_؛

    * `أوامر Symfony Console`_؛

    * `ورقة الغش الخاصة بـ Symfony Console`_.

.. _`مستندات مكون Scheduler`: https://symfony.com/doc/current/scheduler.html
.. _`صياغة Cron/crontab`: https://en.wikipedia.org/wiki/Cron
.. _`مستودع Croncape`: https://github.com/symfonycorp/croncape
.. _`أوامر Symfony Console`: https://symfony.com/doc/current/console.html
.. _`ورقة الغش الخاصة بـ Symfony Console`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
