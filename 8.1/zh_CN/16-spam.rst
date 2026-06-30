用 AI 来阻止垃圾信息
==========================

.. index::
    single: Spam

任何人都可以提交反馈，甚至机器人、垃圾信息发送者等等。我们可以给表单加上某种“验证码”来一定程度上防范机器人，或者也可以使用一些第三方 API。

我决定用一个大语言模型来判断一条评论是否为垃圾信息，以此来演示如何在 Symfony 应用中使用 AI，以及如何把这种代价高昂的调用放到“带外”执行。

获取一个 AI 的 API 秘钥
------------------------------

.. index::
    single: AI
    single: OpenAI

Symfony AI 支持很多模型提供商：OpenAI、Anthropic、Google Gemini、Mistral，甚至支持通过 Ollama 使用本地模型。本章使用 OpenAI：在 `platform.openai.com`_ 上注册并创建一个 API 秘钥。如果你更喜欢另一个提供商，代码保持不变，只是配置会有所不同。

依赖 Symfony AI Bundle
------------------------------

.. index::
    single: Components;AI
    single: AI;Agent
    single: AI;Platform

我们不会自己去调用模型的 HTTP API，而是会使用 Symfony AI Bundle。它为模型提供商提供了一个 *platform* 抽象（每个提供商都作为它自己的桥接包提供），还提供了一个 *agent*，它封装了一个模型来发起调用；而且它也能受益于所有的 Symfony 调试工具，比如和 Symfony 分析器的集成：

.. code-block:: terminal

    $ symfony composer req symfony/ai-bundle symfony/ai-agent symfony/ai-open-ai-platform

.. note::

    Symfony AI 是一套年轻且仍处于实验阶段的组件：它的 API 可能比 Symfony 的其它部分演化得更快。

OpenAI 桥接包的 recipe 已经为我们配置好了 platform；它引用了一个 ``OPENAI_API_KEY`` 环境变量（并在 ``.env`` 中为它添加了一个空的默认值）：

.. code-block:: yaml
    :caption: config/packages/ai_open_ai_platform.yaml
    :class: ignore

    ai:
        platform:
            openai:
                api_key: '%env(OPENAI_API_KEY)%'

在它之上配置一个默认的 *agent*：

.. code-block:: yaml
    :caption: config/packages/ai.yaml

    ai:
        agent:
            default:
                platform: 'ai.platform.openai'
                model: 'gpt-5-mini'

使用环境变量
------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

我们当然不想把秘钥的值硬编码在配置里；这就是为什么它要从 ``OPENAI_API_KEY`` 这个环境变量来读取。

接下来就由每个开发者来决定，是设置一个“真正”的环境变量，还是把这个值存放在一个 ``.env.local`` 文件里：

.. code-block:: text
    :caption: .env.local
    :class: ignore

    OPENAI_API_KEY=sk-...

对于生产环境，应该定义一个“真正”的环境变量。

这样做没有问题，但管理很多环境变量可能会变得繁琐。在这种情况下，对于存储机密信息，Symfony 有一个“更好”的选择。

存储机密信息
------------------

.. index::
    single: Secret

Symfony 可以管理用来存储许多机密信息的 *保险箱*，这可以取代使用很多环境变量。它的一个重要功能是可以把保险箱提交到代码仓库（但是不提交用来解密的 Key 文件）。另一个很棒的功能是每个环境可以有自己的保险箱。

.. index:: ! Command;secrets:set

机密信息就是伪装的环境变量。

把 OpenAI 的 API 秘钥装进保险箱：

.. code-block:: terminal
    :class: answers(OPENAI_API_KEY_VALUE)

    $ symfony console secrets:set OPENAI_API_KEY

.. code-block:: text
    :class: ignore

     Please type the secret value:
     >

     [OK] Secret "OPENAI_API_KEY" encrypted in "config/secrets/dev/"; you can commit it.

因为这是我们第一次运行这个命令，它会在 ``config/secrets/dev/`` 目录下生成两个秘钥。然后它会把 ``OPENAI_API_KEY`` 这个机密信息也存放在同样的目录下。

对于开发环境下的机密信息，你可以决定来把这个保险箱以及 ``config/secrets/dev/`` 目录下生成的秘钥提交到代码仓库。

可以通过设置同名的环境变量来覆盖机密信息的值。

.. index::
    single: Command;secrets:reveal

要从保险箱里读回一个机密信息，使用 ``secrets:reveal``：

.. code-block:: terminal

    $ symfony console secrets:reveal OPENAI_API_KEY

设计一个垃圾信息检查器类
---------------------------------

.. index::
    single: AI;Prompt

在 ``src/`` 下新建一个名为 ``SpamChecker`` 的类，用它来封装询问模型一条评论是否为垃圾信息的逻辑：

.. code-block:: php
    :caption: src/SpamChecker.php

    namespace App;

    use App\Entity\Comment;
    use Symfony\AI\Agent\AgentInterface;
    use Symfony\AI\Platform\Exception\ExceptionInterface;
    use Symfony\AI\Platform\Message\Message;
    use Symfony\AI\Platform\Message\MessageBag;

    class SpamChecker
    {
        public function __construct(
            private AgentInterface $agent,
        ) {
        }

        /**
         * @return int Spam score: 0: not spam, 1: maybe spam, 2: blatant spam
         */
        public function getSpamScore(Comment $comment, array $context): int
        {
            $messages = new MessageBag(
                Message::forSystem(<<<PROMPT
                    You moderate comments submitted to a conference guestbook.
                    Classify the comment as "ham", "maybe spam", or "blatant spam".
                    Only answer with the classification.
                    PROMPT),
                Message::ofUser(sprintf(<<<COMMENT
                    IP: %s
                    User agent: %s
                    Author: %s (%s)
                    Comment: %s
                    COMMENT,
                    $context['user_ip'] ?? '',
                    $context['user_agent'] ?? '',
                    $comment->getAuthor(),
                    $comment->getEmail(),
                    $comment->getText(),
                )),
            );

            try {
                $answer = strtolower($this->agent->call($messages)->getContent());
            } catch (ExceptionInterface) {
                // when the model cannot answer, let a human moderate the comment
                return 1;
            }

            return match (true) {
                str_contains($answer, 'blatant spam') => 2,
                str_contains($answer, 'maybe spam') => 1,
                default => 0,
            };
        }
    }

*系统提示词（system prompt）* 告诉模型它的角色，并约束它的回答；*用户消息（user message）* 包含评论及其提交的上下文（IP 地址、user agent）。

``getSpamScore()`` 方法根据模型的回答返回 3 个值：

* ``2``：如果评论是“明显的垃圾信息”；

* ``1``：如果评论可能是垃圾信息，或者无法连接到模型；

* ``0``：如果评论不是垃圾信息（ham）。

模型的输出是自由文本，即便提示词约束了它：要宽松地解析它（转成小写，使用 ``str_contains()``）。而当模型完全无法回答时，回退到由人来审核，而不是直接报错失败：AI 应该帮助管理员，而绝不应该阻塞留言本。

.. tip::

    试着提交一条看起来明显像垃圾信息的评论，比如 "Buy cheap watches at http://example.com/!!!"，来看看模型是如何工作的。

测试评论是否为垃圾信息
---------------------------------

当有新评论提交时，需要对它进行检查，最好的时机就是在把它存储到数据库之前，调用垃圾信息检查器做这个检查：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -7,6 +7,7 @@ use App\Entity\Conference;
     use App\Form\CommentType;
     use App\Repository\CommentRepository;
     use App\Repository\ConferenceRepository;
    +use App\SpamChecker;
     use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
    @@ -34,7 +35,8 @@ final class ConferenceController extends AbstractController
             Request $request,
             Conference $conference,
             CommentRepository $commentRepository,
    +        SpamChecker $spamChecker,
             #[Autowire('%photo_dir%')] string $photoDir,
             #[MapQueryParameter(options: ['min_range' => 0])] int $offset = 0,
         ): Response {
             $comment = new Comment();
    @@ -48,6 +50,17 @@ final class ConferenceController extends AbstractController
                 }

                 $this->entityManager->persist($comment);
    +
    +            $context = [
    +                'user_ip' => $request->getClientIp(),
    +                'user_agent' => $request->headers->get('user-agent'),
    +                'referrer' => $request->headers->get('referer'),
    +                'permalink' => $request->getUri(),
    +            ];
    +            if (2 === $spamChecker->getSpamScore($comment, $context)) {
    +                throw new \RuntimeException('Blatant spam, go away!');
    +            }
    +
                 $this->entityManager->flush();

                 return $this->redirectToRoute('conference', ['slug' => $conference->getSlug()]);

检查程序是否正常工作。

对评论提交进行速率限制
---------------------------------

.. index::
    single: Rate Limiter
    single: Components;RateLimiter

检测垃圾信息可以保护网站免受老练的垃圾信息发送者的侵扰。一个互补且便宜得多的保护方法，是限制同一个客户端提交评论的速度：没有正常用户会在留言本上一小时发几十条评论。

加入 Symfony 的 Rate Limiter 组件：

.. code-block:: terminal

    $ symfony composer req rate-limiter

配置一个限流器，让它接受同一个客户端每小时最多 5 条评论：

.. code-block:: yaml
    :caption: config/packages/rate_limiter.yaml

    framework:
        rate_limiter:
            comment_submission:
                policy: 'fixed_window'
                limit: 5
                interval: '1 hour'

    when@test:
        framework:
            rate_limiter:
                comment_submission:
                    limit: 1000

自动化测试会在很短时间内合法地提交很多评论，所以 ``test`` 环境下的限额被提高了。

用 ``#[RateLimit]`` 属性在评论提交上强制执行限流器；默认情况下，它通过客户端的 IP 地址来识别客户端：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    +use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
    @@ -31,6 +32,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(
             Request $request,

注意 ``methods`` 参数：浏览一个会议页面是一个 ``GET`` 请求，绝不能被限流；只有评论提交（``POST`` 请求）才被限流。

当达到限额时，Symfony 会自动返回一个 ``429 Too Many Requests`` 应答，并带上一个 ``Retry-After`` HTTP 头，告诉客户端何时可以重试。

同一个组件也能保护管理员登录表单免受暴力破解攻击；在防火墙上启用 *登录限流（login throttling）* 只需一行：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -19,6 +19,7 @@ security:
             main:
                 lazy: true
                 provider: app_user_provider
    +            login_throttling: ~
                 form_login:
                     login_path: app_login
                     check_path: app_login

默认情况下，对于同一个用户名，在一分钟内连续 5 次登录失败后，Symfony 会封禁该 IP（一次成功登录会重置计数）。使用 ``max_attempts`` 和 ``interval`` 选项来调整这个策略。

在生产环境中管理机密信息
------------------------------------

.. index::
    single: Upsun;Secret
    single: Upsun;Environment Variable
    single: Secret
    single: Symfony CLI;cloud:variable:create

在生产环境中，Upsun 支持设置 *敏感环境变量*：

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:OPENAI_API_KEY --value=sk-abcdef

正如以上讨论的，使用 Symfony 的机密信息管理会更好。这并不是从安全性上考虑，而是从项目团队对于机密信息的管理上来考虑。所有的机密信息存放在代码仓库中，你唯一需要管理的生产环境所需的环境变量就是解密秘钥。这样团队中的任何人都可以增加生产环境中的机密信息，即便他们无权访问生产服务器。不过对此的设置会更复杂些。

.. index::
    single: Command;secrets:generate-keys

首先，为生产环境生成一对秘钥：

.. code-block:: terminal

    $ symfony console secrets:generate-keys --env=prod

.. note::

    在 Linux 和类似的操作系统上，使用 ``APP_RUNTIME_ENV=prod`` 代替
    ``--env=prod``，这样可以避免为 ``prod`` 环境编译应用：

    .. code-block:: terminal
        :class: ignore

        $ APP_RUNTIME_ENV=prod symfony console secrets:generate-keys

.. index::
    single: Command;secrets:set

把 OpenAI 的 API 秘钥加入生产环境的保险箱中，但这次是用针对生产环境的值：

.. code-block:: terminal
    :class: answers(sk-abcdef)

    $ symfony console secrets:set OPENAI_API_KEY --env=prod

最后一步是通过设置一个敏感环境变量，把解密秘钥发送给 Upsun：

.. code-block:: terminal

    $ symfony cloud:variable:create --sensitive=1 --level=project -y --name=env:SYMFONY_DECRYPTION_SECRET --value=`php -r 'echo base64_encode(include("config/secrets/prod/prod.decrypt.private.php"));'`

你可以添加和提交所有文件；解密秘钥已经被自动加到 ``.gitignore`` 里了，所以它永远不会被提交。既然已经部署完了，出于安全考虑，你可以将它从你的本地电脑中移除：

.. code-block:: terminal

    $ rm -f config/secrets/prod/prod.decrypt.private.php

.. sidebar:: 深入学习

    * `Symfony AI 文档`_；

    * `环境变量处理器`_；

    * `如何对敏感信息保密`_。

.. _`platform.openai.com`: https://platform.openai.com
.. _`Symfony AI 文档`: https://symfony.com/doc/current/ai/index.html
.. _`环境变量处理器`: https://symfony.com/doc/current/configuration/env_var_processors.html
.. _`如何对敏感信息保密`: https://symfony.com/doc/current/configuration/secrets.html
