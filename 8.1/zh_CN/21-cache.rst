使用缓存提升性能
========================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

性能问题可能常常会出现。一些典型例子：缺少数据库索引、每个页面上太多的 SQL 请求。在空的数据库里，你不会有任何问题，但随着流量和数据量的增长，问题可能会在某个时刻显现出来。

增加 HTTP 缓存头
---------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

使用 HTTP 的缓存策略是一个最大化提升用户性能的好方案，而且它不用花费很多代价。为了更好的性能，可以在生产服务器上增加反向代理的缓存，也可以用 `CDN`_ 在靠近用户的节点上部署缓存。

让我们来把首页缓存一小时：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -14,6 +14,7 @@ use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\DependencyInjection\Attribute\Autowire;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\Attribute\Cache;
     use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
     use Symfony\Component\HttpKernel\Attribute\RateLimit;
     use Symfony\Component\Messenger\MessageBusInterface;
    @@ -27,6 +28,7 @@ final class ConferenceController extends AbstractController
         ) {
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/', name: 'homepage')]
         public function index(ConferenceRepository $conferenceRepository): Response
         {

``#[Cache]`` 属性通过它的 ``smaxage`` 参数来配置反向代理的缓存过期时间；用 ``maxage`` 来控制浏览器缓存。时间单位是秒（1 小时=60 分钟=3600 秒）。和路由或速率限制一样，缓存策略就声明在它生效的地方：控制器上。

缓存会议页面更有挑战一些，因为它的内容更加动态。任何人在任何时候都可以增加一条评论，没有人想要等一个小时才能看到它发布。在这种情况下，使用 *HTTP 验证* 的策略。

激活 Symfony 的 HTTP 缓存内核
------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

为了测试 HTTP 缓存策略，我们来启用 Symfony 的 HTTP 反向代理，但只在“开发”环境中（对于“生产”环境，我们会使用一个“更健壮”的方案）：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -22,3 +22,7 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    +
    +when@dev:
    +    framework:
    +        http_cache: true

Symfony 的 HTTP 反向代理（由 ``HttpCache`` 类实现）不但是一个功能完善的 HTTP 反向代理，也以 HTTP 头的形式增加了友好的调试信息。这对验证我们已经设置的缓存头很有帮助。

在首页上检查缓存：

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store
    content-length: 50978

对于第一次请求，缓存服务器告诉你这是个 ``miss`` （未命中），而且它执行了 ``store`` （存储）来把应答缓存起来。检查 ``cache-control`` 头可以看到配置的缓存策略。

对于后续的请求，应答都是来自缓存（ ``age`` 信息也会被更新）：

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 143
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:11:57 GMT
    x-content-digest: en63cef7045fe418859d73668c2703fb1324fcc0d35b21d95369a9ed1aca48e73e
    x-debug-token: 9eb25a
    x-debug-token-link: https://127.0.0.1:8000/_profiler/9eb25a
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh
    content-length: 50978

用 ESI 来避免 SQL 请求
----------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

``TwigEventListener`` 事件监听器会在 Twig 里注入一个全局变量，该变量包含了所有的会议对象。网站的每一页里都会注入该变量。这很可能是一个优化的好目标。

你不会每天都去添加会议，所以代码总是一次次地从数据库里去查询完全相同的数据。

我们可能想要用 Symfony 来缓存会议的名字和 slug，但我喜欢尽可能地依靠 HTTP 缓存提供的机制。

当你想要缓存页面的一个片段时，通过创建一个 *子请求* 来把该片段从当前 HTTP 请求中移出。在这种使用场景下， *ESI* 是一个完美的搭配。一个 ESI 可以把一个 HTTP 请求的结果嵌入到另一个请求的结果。

创建一个控制器，让它只返回展示会议列表的 HTML 片段。

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,14 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Route('/conference_header', name: 'conference_header')]
    +    public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
    +    {
    +        return $this->render('conference/header.html.twig', [
    +            'conferences' => $conferenceRepository->findAll(),
    +        ]);
    +    }
    +
         #[RateLimit('comment_submission', methods: ['POST'])]
         #[Route('/conference/{slug:conference}', name: 'conference')]
         public function show(

创建相应的模板：

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

点击 ``/conference_header`` 来检查一切是否正常工作。

.. index::
    single: Twig;render
    single: Twig;path

揭秘戏法的时候到了！通过更新 Twig 布局来调用我们刚刚创建的控制器：

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,11 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            <ul>
    -            {% for conference in conferences %}
    -                <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
    -            {% endfor %}
    -            </ul>
    +            {{ render(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

*好了*。刷新下页面，网站的显示内容没有变化。

.. tip::

    使用 Symfony 分析器的 "Request / Response" 面板来更多了解主请求和子请求。

现在，每次你在浏览器里打开一个页面，会执行两个 HTTP 请求，一个是针对页头，另一个是针对页面主体。你把性能变得更差了。恭喜你！

现在，针对会议页头的 HTTP 调用是在 Symfony 内部完成的，所以不会产生 HTTP 的回路。这也意味着它没有办法从 HTTP 缓存头中获益。

用 ESI 把这个调用转换成一个“真正”的 HTTP 调用。

首先，启用 ESI 支持：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -12,7 +12,7 @@ framework:
             cookie_secure: auto
             cookie_samesite: lax

    -    #esi: true
    +    esi: true
         #fragments: true
         php_errors:
             log: true

.. index::
    single: Twig;render_esi
    single: Twig;path

然后，使用 ``render_esi`` 来代替 ``render``：

.. code-block:: diff
    :caption: patch_file

    --- i/templates/base.html.twig
    +++ w/templates/base.html.twig
    @@ -14,7 +14,7 @@
         <body>
             <header>
                 <h1><a href="{{ path('homepage') }}">Guestbook</a></h1>
    -            {{ render(path('conference_header')) }}
    +            {{ render_esi(path('conference_header')) }}
                 <hr />
             </header>
             {% block body %}{% endblock %}

如果 Symfony 检测到一个会处理 ESI 的反向代理，它就会自动启用对它的支持（如果没有检测到，它就会回退到同步渲染子请求）。

由于 Symfony 的反向代理的确支持 ESI，我们来查看下日志（先清空缓存，请看下面的“清理缓存”）：

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 0
    cache-control: must-revalidate, no-cache, private
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 08:20:05 GMT
    expires: Mon, 28 Oct 2019 08:20:05 GMT
    x-content-digest: en4dd846a34dcd757eb9fd277f43220effd28c00e4117bed41af7f85700eb07f2c
    x-debug-token: 719a83
    x-debug-token-link: https://127.0.0.1:8000/_profiler/719a83
    x-robots-tag: noindex
    x-symfony-cache: GET /: miss, store; GET /conference_header: miss
    content-length: 50978

刷新几次：``/`` 的应答被缓存了，``/conference_header`` 没有被缓存。我们已经取得了很好的成果：总的页面在缓存中，但有一部分仍然是动态的。

但这仍不是我们想要的。我们来把 ``/conference_header`` 缓存一小时，并且独立于其它部分：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -36,6 +36,7 @@ final class ConferenceController extends AbstractController
             ]);
         }

    +    #[Cache(smaxage: 3600)]
         #[Route('/conference_header', name: 'conference_header')]
         public function conferenceHeader(ConferenceRepository $conferenceRepository): Response
         {

现在两个请求都被缓存了：

.. code-block:: terminal
    :class: ignore

    $ curl -s -I -X GET https://127.0.0.1:8000/

.. code-block:: text
    :class: ignore
    :emphasize-lines: 2,3,10

    HTTP/2 200
    age: 613
    cache-control: public, s-maxage=3600
    content-type: text/html; charset=UTF-8
    date: Mon, 28 Oct 2019 07:31:24 GMT
    x-content-digest: en15216b0803c7851d3d07071473c9f6a3a3360c6a83ccb0e550b35d5bc484bbd2
    x-debug-token: cfb0e9
    x-debug-token-link: https://127.0.0.1:8000/_profiler/cfb0e9
    x-robots-tag: noindex
    x-symfony-cache: GET /: fresh; GET /conference_header: fresh
    content-length: 50978

``x-symfony-cache`` 头包含两个元素：``/`` 主请求和一个子请求（``conference_header`` 这个 ESI）。两者都在缓存中（``fresh``）。

主页面和它的 ESI 可以采用不同的缓存策略。如果我们有一个”关于“页面，我们可能想要在缓存中让它存放一星期的时间，但仍然让页头每小时更新一次。

移除事件监听器，我们不再需要它了：

.. code-block:: terminal

    $ rm src/EventListener/TwigEventListener.php

为测试而清空 HTTP 缓存
------------------------------

增加了缓存层之后，在浏览器中或用一些自动化测试手段来测试网站变得更困难一些了。

你可以通过删除 ``var/cache/dev/http_cache/`` 目录来手工移除所有的 HTTP 缓存：

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

如果你只是想让一部分 URL 的缓存失效，或者你想要在功能测试中让缓存失效，那这个策略就不太可行。让我们来增加一个小的管理员专用的 HTTP 地址，用来让某些 URL 的缓存失效：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/security.yaml
    +++ w/config/packages/security.yaml
    @@ -20,6 +20,8 @@ security:
                     login_path: app_login
                     check_path: app_login
                     enable_csrf: true
    +            http_basic: { realm: Admin Area }
    +            entry_point: form_login
                 logout:
                     path: app_logout
                     # where to redirect after logout
    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -8,6 +8,8 @@ use Doctrine\ORM\EntityManagerInterface;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
    +use Symfony\Component\HttpKernel\HttpCache\StoreInterface;
    +use Symfony\Component\HttpKernel\KernelInterface;
     use Symfony\Component\Messenger\MessageBusInterface;
     use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
    @@ -47,4 +49,16 @@ class AdminController extends AbstractController
                 'comment' => $comment,
             ]));
         }
    +
    +    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
    +    {
    +        if ('prod' === $kernel->getEnvironment()) {
    +            return new Response('KO', 400);
    +        }
    +
    +        $store->purge($request->getSchemeAndHttpHost().'/'.$uri);
    +
    +        return new Response('Done');
    +    }
     }

新的控制器被限制为只能用 HTTP 的 ``PURGE`` 方法访问。这个方法并不是 HTTP 标准中的，但被广泛用于让缓存失效。

默认情况下，路由参数里不能包含 ``/`` 字符，因为它是用来为 URL 分段的。但在最后一个路由参数里，你可以覆盖这个限制，比如在 ``uri`` 参数里，你可以设置自己的需求匹配模式（``.*``）。

我们获得 ``HttpCache`` 实例的方式看上去也有点奇怪；我们在用一个匿名函数，因为无法访问“真正”的类。 ``HttpCache`` 的实例包含了真正的内核，而内核并不知道这个缓存层的存在，它也不应该知道。

用下面的 cURL 命令来让首页和会议页头的缓存失效：

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` 子命令会返回本地 web 服务器的当前 URL。

.. note::

    因为代码不会引用这个控制器，所以它没有路由名。

在开发环境中停用 HTTP 缓存
---------------------------------------

HTTP 缓存对于验证我们的缓存头以及学习如何清理过期条目很有用。但在开发环境中启用一个反向代理是不寻常的，而且它很快就会带来麻烦：当你迭代代码时，应答会从缓存里返回，由于 HttpCache 在处理文件应答时一个长期存在的限制，一些 vendor 资源甚至会以空 body 返回。

现在一切都已验证完毕，把它停用；在生产环境中 Varnish 会接管这个工作：

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -14,7 +14,3 @@ when@test:
             test: true
             session:
                 storage_factory_id: session.storage.factory.mock_file
    -
    -when@dev:
    -    framework:
    -        http_cache: true

为相似的路由加上前缀并归为一组
---------------------------------------------

.. index::
    single: Attributes;Route

管理后台控制器里的两个路由都有相同的 ``/admin`` 前缀。重构这些路由，改为在类上设置这个前缀，而不是在所有路由上重复它：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -15,6 +15,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -24,7 +25,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, WorkflowInterface $commentStateMachine): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -50,7 +51,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

缓存 CPU/内存密集型的操作
-----------------------------------

.. index::
    single: Process
    single: Components;Process

在这个网站上，我们没有 CPU 或内存密集型的算法。为了讨论下”本地缓存“，我们来创建一个命令，它会展示我们当前的工作步骤（更准确地说，就是在当前的 Git 提交时附带的 Git 标签）。

Symfony 的 Process 组件可以用来运行一个命令，并且获得命令的输出结果（标准输出和错误输出）。

实现这个命令：

.. code-block:: php
    :caption: src/Command/StepInfoCommand.php

    namespace App\Command;

    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Process\Process;

    #[AsCommand('app:step:info')]
    class StepInfoCommand
    {
        public function __invoke(OutputInterface $output): int
        {
            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
            $process->mustRun();
            $output->write($process->getOutput());

            return Command::SUCCESS;
        }
    }

.. index::
    single: Cache
    single: Components;Cache

如果你想要把输出的内容缓存几分钟，那要怎么做？使用 Symfony 缓存。

Symfony 会把命令的 ``__invoke()`` 方法里类型提示的服务注入进来，方式和控制器参数一样。用缓存逻辑包装这段代码：

.. code-block:: diff
    :caption: patch_file

    --- i/src/Command/StepInfoCommand.php
    +++ w/src/Command/StepInfoCommand.php
    @@ -6,15 +6,21 @@ use Symfony\Component\Console\Attribute\AsCommand;
     use Symfony\Component\Console\Command\Command;
     use Symfony\Component\Console\Output\OutputInterface;
     use Symfony\Component\Process\Process;
    +use Symfony\Contracts\Cache\CacheInterface;

     #[AsCommand('app:step:info')]
     class StepInfoCommand
     {
    -    public function __invoke(OutputInterface $output): int
    +    public function __invoke(OutputInterface $output, CacheInterface $cache): int
         {
    -        $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    -        $process->mustRun();
    -        $output->write($process->getOutput());
    +        $step = $cache->get('app.current_step', function ($item) {
    +            $process = new Process(['git', 'tag', '-l', '--points-at', 'HEAD']);
    +            $process->mustRun();
    +            $item->expiresAfter(30);
    +
    +            return $process->getOutput();
    +        });
    +        $output->writeln($step);

             return Command::SUCCESS;
         }

现在，只有当 ``app.current_step`` 项没有被缓存时，那个命令才会执行。

分析并比较性能
---------------------

绝不要盲目地增加缓存。记住，增加一些缓存就增加了一层复杂度。而且因为我们往往猜不准什么会变快什么会变慢，所以我们可能最终会发现缓存反而让应用变慢了。

总要去衡量缓存带来的性能影响，可以使用类似 `Blackfire`_ 的分析工具。

参考关于“性能”章节的内容来更多地了解如何在部署前使用 Blackfire 测试代码。

在生产环境中配置反向代理缓存
------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

我们不再在生产环境中使用 Symfony 的反向代理，而是使用“更健壮”的 Varnish 反向代理。

在 Upsun 服务中加入 Varnish：

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -6,6 +6,15 @@ services:
         database:
             type: postgresql:16

    +    varnish:
    +        type: varnish:9.0
    +        relationships:
    +            application: 'app:http'
    +        configuration:
    +            vcl: !include
    +                type: string
    +                path: config.vcl
    +
     applications:

.. index::
    single: Upsun;Routes

用 Varnish 作为路由的主入口：

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

最后，新建一个 ``config.vcl`` 文件来配置 Varnish：

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

在 Varnish 上启用 ESI 支持
--------------------------------

需要为每个请求显式启用 Varnish 上的 ESI 支持。为了使它具有通用性，Symfony 使用标准的 ``Surrogate-Capability`` 和 ``Surrogate-Control`` 头来协商 ESI 支持：

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
        set req.http.Surrogate-Capability = "abc=ESI/1.0";
    }

    sub vcl_backend_response {
        if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
            unset beresp.http.Surrogate-Control;
            set beresp.do_esi = true;
        }
    }

清理 Varnish 缓存
---------------------

在生产环境中很可能永远不需要使缓存失效，除非是为了处理紧急情况，或者可能是在非 ``master`` 分支上。如果你需要经常去清理缓存，它很可能意味着你的缓存策略需要调整（减少缓存有效时间，或者是用验证策略代替过期策略）。

不管怎样，我们来看下如何配置 Varnish 的缓存失效：

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,6 +1,13 @@
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    +
    +    if (req.method == "PURGE") {
    +        if (req.http.x-purge-token != "PURGE_NOW") {
    +            return(synth(405));
    +        }
    +        return (purge);
    +    }
     }

     sub vcl_backend_response {

在真实场景下，你很可能会根据IP来限制访问，`Varnish 文档`_ 里有这方面的介绍。

现在清理一些 URL 缓存：

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

这些 URL 看起来有点奇怪，因为 ``env:urls`` 返回的URL已经以 ``/`` 结尾了。

.. sidebar:: 深入学习

    * `Cloudflare`_，覆盖全球的云平台；

    * `Varnish 的 HTTP 缓存文档`_；

    * `ESI 规范`_ 和 `ESI 开发者资源`_；

    * `HTTP 缓存有效模型`_；

    * `Upsun中的HTTP缓存`_。

.. _`Blackfire`: https://blackfire.io/
.. _`Varnish 文档`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Varnish 的 HTTP 缓存文档`: https://varnish-cache.org/docs/index.html
.. _`ESI 规范`: https://www.w3.org/TR/esi-lang
.. _`ESI 开发者资源`: https://www.akamai.com/us/en/support/esi.jsp
.. _`HTTP 缓存有效模型`: https://symfony.com/doc/current/http_cache/validation.html
.. _`Upsun中的HTTP缓存`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
