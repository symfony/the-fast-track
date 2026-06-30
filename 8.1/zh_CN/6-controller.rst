创建控制器
===============

.. index::
    single: Controller
    single: Routing;Route

我们的留言本项目已经在生产服务器上了，但我们做了点弊。这个项目还没有任何网页。首页只不过是个无聊的 404 错误页面。让我们来修复它。

当一个 HTTP 请求进来的时候，比如访问首页的时候（``http://localhost:8000/``），Symfony 试着去找到一条匹配 *请求路径* 的 *路由* （在本例中是请求路径 ``/``）。一条 *路由* 就是一个请求路径和一个 *PHP callable* 之间的链接，而这个 *PHP callable* 是一个函数，它为该请求生成一个 HTTP *应答*。

这些 *callable* 称为“控制器”。在 Symfony 里，大部分控制器都以 PHP 类的形式来实现。你可以手工创建这样的一个类，但是因为我们喜欢快速开发，我们来看看 Symfony 如何能帮助我们。

用 *Maker Bundle* 来偷偷懒
-------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

我们可以用 ``symfony/maker-bundle`` 包来轻松生成控制器，它作为 ``webapp`` 包的一部分已经安装好了。

maker bundle 帮你生成很多不同的类。我们在本书中会一直用到它。每一个”生成器“对应一个命令，所有这些命令都是 ``make`` 命令命名空间的一部分。

.. index::
    single: Command;list

Symfony Console 自带 ``list`` 命令，它会列出某个命名空间下的所有可用命令；你用它来查看 maker bundle 提供的所有生成器：

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

选择配置的格式
---------------------

在创建项目的第一个控制器之前，我们需要选择配置的格式。Symfony 自带支持 YAML、XML、PHP 和 PHP 属性（attribute）。

对于 *和依赖包相关的配置*，*YAML* 是最好的选择。这是 ``config/`` 目录里的文件所使用的格式。一种常见的情况是，当你安装一个新的包时，包的 *recipe* 会添加一个 ``.yaml`` 后缀的文件到 ``config/`` 目录下。

对于 *和 PHP 代码相关的配置*，*属性* 是更好的选择，因为它们就定义在代码旁边。让我用一个例子来解释下。当一个请求进来，需要有一些配置去告诉 Symfony 该请求路径需要被哪个具体的控制器（一个 PHP 类）来处理。当使用 YAML、XML 或者 PHP 这些配置格式的时候，涉及到两个文件（配置文件和 PHP 控制器文件）。当使用属性时，配置是直接写在控制器类里的。

你可能会想，当要为某个功能安装一个包时，该如何知道包的名字？大多数时候，你不必知道。在很多情况下，Symfony 会在错误信息中指出要安装的包。举个例子，如果运行 ``symfony console make:message`` 命令时还没有安装 ``messenger`` 包，那异常信息就会显示，它会提示你所需要安装的包。

生成一个控制器
---------------------

.. index::
    single: Command;make:controller

用 ``make:controller`` 命令来创建你的第一个 *控制器*：

.. code-block:: terminal
    :class: answers(no)

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

这个命令在 ``src/Controller/`` 目录下新建了一个名为 ``ConferenceController`` 的类。这个生成的类由一些样本代码组成，你可以去微调它：

.. code-block:: php
    :caption: src/Controller/ConferenceController.php
    :class: ignore
    :emphasize-lines: 9

    namespace App\Controller;

    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;

    final class ConferenceController extends AbstractController
    {
        #[Route('/conference', name: 'app_conference')]
        public function index(): Response
        {
            return $this->render('conference/index.html.twig', [
                'controller_name' => 'ConferenceController',
            ]);
        }
    }

``#[Route('/conference', name: 'app_conference')]`` 属性使 ``index()`` 方法成为一个控制器（这段配置就在它所配置的代码旁边）。

当你用浏览器访问 ``/conference`` 路径时，这个控制器就会被执行，它返回一个应答。

调整一下路由，让它匹配首页的路径：

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 7

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -8,7 +8,7 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/conference', name: 'app_conference')]
    +    #[Route('/', name: 'homepage')]
         public function index(): Response
         {
             return $this->render('conference/index.html.twig', [

当我们想要在代码里引用首页的地址时，路由的 ``name`` 属性会被用到。我们不会硬编码 ``/`` 路径，而是会用路由名。

我们来返回一个简单的 HTML 页面，来取代这个默认的页面：

.. code-block:: diff
    :caption: patch_file
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -11,8 +11,13 @@ final class ConferenceController extends AbstractController
         #[Route('/', name: 'homepage')]
         public function index(): Response
         {
    -        return $this->render('conference/index.html.twig', [
    -            'controller_name' => 'ConferenceController',
    -        ]);
    +        return new Response(<<<EOF
    +            <html>
    +                <body>
    +                    <img src="/images/under-construction.gif" />
    +                </body>
    +            </html>
    +            EOF
    +        );
         }
     }

刷新浏览器：

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

控制器的主要责任就是针对请求返回一个 HTTP ``Response`` 对象。

本章接下来的内容都是我们不会保留的代码，所以现在先提交我们的改动：

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

加一个彩蛋
---------------

为了演示应答如何来利用请求中的信息，让我们来加一个小的 `彩蛋`_。当首页路径包含了一个类似 ``?hello=Fabien`` 的查询字符串时，让我们添加一些文字来对字符串中的人致意：

.. code-block:: diff
    :emphasize-lines: 18

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,17 +3,24 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    +    public function index(Request $request): Response
         {
    +        $greet = '';
    +        if ($name = $request->query->get('hello')) {
    +            $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
    +        }
    +
             return new Response(<<<EOF
                 <html>
                     <body>
    +                    $greet
                         <img src="/images/under-construction.gif" />
                     </body>
                 </html>

Symfony 通过 ``Request`` 对象来暴露请求中的数据。当 Symfony 看到控制器里有这个类型提示的参数，它会自动把它传递给你。我们可以用它来获得查询字符串中的名字，然后添加一个 ``<h1>`` 标题。

现在浏览器里打开 ``/`` 路径，再打开 ``/?hello=Fabien`` 路径，看一下两者的不同。

.. note::

    注意一下对 ``htmlspecialchars()`` 函数的调用，是为了防止 XSS 攻击。当我们切换到一个良好的模板引擎后，这些都会自动为我们完成。

我们也可以把这个人的名字作为 URL 的一部分：

.. code-block:: diff

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -9,11 +9,11 @@ use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
    -    #[Route('/', name: 'homepage')]
    -    public function index(Request $request): Response
    +    #[Route('/hello/{name}', name: 'homepage')]
    +    public function index(string $name = ''): Response
         {
             $greet = '';
    -        if ($name = $request->query->get('hello')) {
    +        if ($name) {
                 $greet = sprintf('<h1>Hello %s!</h1>', htmlspecialchars($name));
             }

路由中的 ``{name}`` 部分是动态 *路由参数*，它类似于通配符。你可以在浏览器先访问 ``/hello`` 再访问 ``/hello/Fabien``，看到的结果和之前一样。在控制器参数里用同 *名* 的参数，能得到 ``{name}`` 参数的 *值*。在本例中，参数名就叫 ``$name``。

撤销我们刚才所做的改动：

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

调试变量
-------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

一个很棒的调试助手是 Symfony 的 ``dump()`` 函数。它始终可用，你可以用它来导出复杂的变量，变量会以美观且可交互式的格式展现出来。

临时更改下 ``src/Controller/ConferenceController.php`` ，让它导出 Request 对象：

.. code-block:: diff
    :emphasize-lines: 17

    --- i/src/Controller/ConferenceController.php
    +++ w/src/Controller/ConferenceController.php
    @@ -3,14 +3,17 @@
     namespace App\Controller;

     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    +use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;

     final class ConferenceController extends AbstractController
     {
         #[Route('/', name: 'homepage')]
    -    public function index(): Response
    -    {
    +    public function index(Request $request): Response
    +        {
    +        dump($request);
    +
             return new Response(<<<EOF
                 <html>
                     <body>

刷新页面后，注意下工具栏里出现了新的“目标”图标，它可以让你检查导出的变量。点击那个图标会打开一个另外的页面，在那里查看导出的变量会更方便：

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

撤销我们刚才所做的改动：

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: 深入学习

    * Symfony `路由`_ 系统；

    * `SymfonyCasts 的路由，控制器和页面教程`_；

    * `PHP 属性`_；

    * `HttpFoundation 组件`_；

    * `XSS （跨站脚本）`_ 攻击；

    * `Symfony 路由速查表`_。

.. _`彩蛋`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`路由`: https://symfony.com/doc/current/routing.html
.. _`SymfonyCasts 的路由，控制器和页面教程`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`PHP 属性`: https://www.php.net/attributes
.. _`HttpFoundation 组件`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS （跨站脚本）`: https://owasp.org/www-community/attacks/xss/
.. _`Symfony 路由速查表`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
