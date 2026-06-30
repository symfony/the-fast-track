为用户界面添加样式
==========================

.. index::
    single: AssetMapper
    single: Components;AssetMapper
    single: Stylesheet

我们还完全没有花时间在用户界面的设计上。为了像专业人士那样添加样式，我们会使用一套基于 *AssetMapper* 的现代技术栈，AssetMapper 是 Symfony 的一个组件，从本书的第一步开始它就一直在管理我们的前端资产。

AssetMapper 拥抱现代 web 标准：JavaScript 和 CSS 文件会原样提供，并通过一个 *importmap* 连接在一起，让浏览器直接加载原生的 *ES 模块*。没有打包器，没有构建步骤，也不需要 Node.js。

看一下项目根目录下的 ``importmap.php`` 文件：它描述了应用程序使用的 JavaScript 包。在 ``templates/base.html.twig`` 里调用的 ``importmap()`` Twig 函数会把它们暴露给浏览器。

利用 Bootstrap
--------------------

.. index::
    single: Bootstrap

要从良好的默认值开始并构建一个响应式网站，像 `Bootstrap`_ 这样的 CSS 框架可以帮上大忙。把它作为一个 importmap 包来安装：

.. code-block:: terminal

    $ symfony console importmap:require bootstrap bootstrap/dist/css/bootstrap.min.css

这个命令会把这个包注册到 ``importmap.php`` 里，并把它（以及它的 ``@popperjs/core`` 依赖）下载到 ``assets/vendor/`` 目录；应用程序在运行时不依赖任何 CDN。

在主 JavaScript 入口里导入 Bootstrap（我们也清理掉了默认的欢迎消息）：

.. code-block:: diff
    :caption: patch_file

    --- i/assets/app.js
    +++ w/assets/app.js
    @@ -5,6 +5,6 @@ import './stimulus_bootstrap.js';
      * This file will be included onto the page via the importmap() Twig function,
      * which should already be in your base.html.twig.
      */
    +import 'bootstrap';
    +import 'bootstrap/dist/css/bootstrap.min.css';
     import './styles/app.css';
    -
    -console.log('This log comes from assets/app.js - welcome to AssetMapper! 🎉');

注意 ``app.css`` 是在 Bootstrap 样式 *之后* 导入的，这样我们的定制就会生效。

Symfony 的表单系统通过一个专门的主题原生支持 Bootstrap，启用它：

.. code-block:: yaml
    :caption: config/packages/twig.yaml

    twig:
        form_themes: ['bootstrap_5_layout.html.twig']

为 HTML 添加样式
----------------

现在我们准备好为应用程序添加样式了。在项目根目录下载并解压这个压缩包：

.. code-block:: terminal

    $ php -r "copy('https://symfony.com/uploads/assets/guestbook-8.1.zip', 'guestbook-8.1.zip');"
    $ unzip -o guestbook-8.1.zip
    $ rm guestbook-8.1.zip

看一下这些模板，你也许能学到一两个关于 Twig 的小技巧。

提供资产
------------------

.. index::
    single: AssetMapper;asset-map:compile

没有什么需要构建的：刷新一个页面，改动就立即生效了。在开发环境中，AssetMapper 会直接提供资产文件。

花点时间来发现这些视觉上的变化。在浏览器里看一下新的设计。

.. figure:: screenshots/design-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

.. figure:: screenshots/design-conference.png
    :alt: /conference/amsterdam-2019
    :align: center
    :figclass: with-browser

生成的登录表单现在也有样式了，因为 Maker bundle 默认使用了 Bootstrap 的 CSS 类：

.. figure:: screenshots/login-styled.png
    :alt: /login
    :align: center
    :figclass: with-browser

对于生产环境，Upsun 会在构建阶段自动运行 ``asset-map:compile`` 命令：所有资产都会被复制到 ``public/assets/`` 目录，文件名中带有一个版本哈希，从而启用安全的、长期的 HTTP 缓存。

.. sidebar:: 深入学习

    * `AssetMapper 组件文档`_；

    * `importmap 规范`_；

    * `Bootstrap 文档`_。

.. _`Bootstrap`: https://getbootstrap.com/
.. _`AssetMapper 组件文档`: https://symfony.com/doc/current/frontend/asset_mapper.html
.. _`importmap 规范`: https://html.spec.whatwg.org/multipage/webappapis.html#import-maps
.. _`Bootstrap 文档`: https://getbootstrap.com/docs/
