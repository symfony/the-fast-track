调度任务
================

.. index::
    single: Cron

有些维护任务必须按计划运行。与持续运行的 worker 不同，被调度的任务会周期性地运行很短的一段时间。

清理评论
--------------------

被标记为垃圾信息或被管理员拒绝的评论会保留在数据库里，因为管理员可能想再检视它们一小段时间。但它们大概在一段时间后就应该被移除。在创建之后保留一星期大概就足够了。

在评论的 repository 里创建一些工具方法，用来查找被拒绝的评论、统计它们的数量并删除它们：

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

    对于更复杂的查询，看一下生成的 SQL 语句有时会很有用（它们可以在日志里找到，对于 Web 请求也可以在分析器里找到）。

使用类常量、容器参数和环境变量
----------------------------------------------------------------------

.. index::
    single: Container;Parameters

7 天？我们也可以选另一个数字，比如 10 或 20。这个数字可能会随时间变化。我们决定把它作为类上的一个常量来存储，但我们也可以把它作为一个参数存储在容器里，甚至可以把它定义为一个环境变量。

下面是一些用来决定使用哪种抽象的经验法则：

* 如果这个值是敏感的（密码、API 令牌等等），使用 Symfony 的 *机密信息存储（secret storage）* 或一个 Vault；

* 如果这个值是动态的，而且你应该能够在 *不* 重新部署的情况下改变它，使用一个 *环境变量*；

* 如果这个值在不同环境之间可以不同，使用一个 *容器参数*；

* 对于其它所有情况，把这个值存储在代码里，比如作为一个 *类常量*。

创建一个命令行命令
----------------------

移除旧评论是适合用 cron 任务来做的完美任务。它应该定期执行，而稍有延迟也不会有什么大的影响。

通过创建一个 ``src/Command/CommentCleanupCommand.php`` 文件来创建一个名为 ``app:comment:cleanup`` 的命令行命令：

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

所有的应用命令都和 Symfony 自带的命令一起注册，它们都可以通过 ``symfony console`` 来访问。由于可用命令的数量可能很大，你应该为它们设置命名空间。按照惯例，应用命令应该存放在 ``app`` 命名空间下。用冒号（``:``）分隔，可以添加任意多层的子命名空间。

一个命令通过在 ``__invoke()`` 的参数上使用 ``#[Argument]`` 和 ``#[Option]`` 属性来声明它的 *参数（arguments）* 和 *选项（options）*（``$dryRun`` 参数变成了 ``--dry-run`` 选项）。Symfony 会根据类型来注入其它参数：``SymfonyStyle`` 用来向控制台写出格式美观的输出，而任何服务（比如评论的 repository）的注入方式和控制器参数一样。

运行这个命令来清理数据库：

.. code-block:: terminal

    $ symfony console app:comment:cleanup

调度这个命令
----------------------

.. index::
    single: Scheduler
    single: Components;Scheduler
    single: Attributes;AsCronTask

手工运行这个命令是可行的，但它应该每晚运行。Symfony 的 Scheduler 组件会按计划生成消息；然后这些消息会被一个 worker 消费，就像其它任何 Messenger 消息一样。

加入 Scheduler 组件，以及解析 cron 表达式的库：

.. code-block:: terminal

    $ symfony composer req scheduler dragonmantank/cron-expression

用 ``#[AsCronTask]`` 属性来调度这个命令：

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

这个属性用一个 cron 表达式把命令注册到默认的 *schedule（计划）* 上：每晚 11:50 pm（UTC）。检查它：

.. code-block:: terminal

    $ symfony console debug:scheduler

一个 schedule 会作为一个以它命名的常规 Messenger 传输暴露出来；像消费任何其它传输一样消费它：

.. code-block:: terminal

    $ symfony run -d symfony console messenger:consume scheduler_default -vv

部署这个计划
----------------------

.. index::
    single: Upsun;Workers

在 Upsun 上，worker 只消费 ``async`` 传输。让它也消费这个计划：

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

只需这么做就够了：没有 crontab，没有额外的进程；这个计划就活在 PHP 代码里，紧挨着它所触发的任务，并且像应用程序的其它部分一样被部署和版本化。

那系统级的 cron 呢？
------------------------

.. index::
    single: Upsun;Cron
    single: Upsun;Croncape

Upsun 也支持操作系统级的 cron 任务，它们和 web 容器以及 worker 一起描述在 ``.upsun/config.yaml`` 里；默认配置已经定义了一个用来清理过期 PHP 会话的 cron。系统 cron 很适合那些不是用 PHP 实现的任务。

默认 cron 所使用的 ``croncape`` 工具会监控命令的执行，如果命令返回任何不为 ``0`` 的退出码，它就会向 ``MAILTO`` 环境变量里定义的地址发送一封邮件：

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:MAILTO --value=ops@example.com

注意，cron 会在所有 Upsun 分支上设置。如果你不想在非生产环境上运行某些 cron，检查 ``$PLATFORM_ENVIRONMENT_TYPE`` 环境变量：

.. code-block:: bash
    :class: ignore

    if [ "$PLATFORM_ENVIRONMENT_TYPE" = "production" ]; then
        croncape symfony app:invoices:send
    fi

.. sidebar:: 深入学习

    * `Scheduler 组件文档`_；

    * `Cron/crontab 语法`_；

    * `Croncape 代码仓库`_；

    * `Symfony Console 命令`_；

    * `Symfony Console 速查表`_。

.. _`Scheduler 组件文档`: https://symfony.com/doc/current/scheduler.html
.. _`Cron/crontab 语法`: https://en.wikipedia.org/wiki/Cron
.. _`Croncape 代码仓库`: https://github.com/symfonycorp/croncape
.. _`Symfony Console 命令`: https://symfony.com/doc/current/console.html
.. _`Symfony Console 速查表`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/console_en_42.pdf
