管理性能
============

.. index::
    single: Blackfire
    single: Profiler

过早的优化是万恶之源。

或许你之前已经听过这句引用的话。但我想要完整引用它：

大概在 97% 的情况下，我们都应该忽略小的性能优化：过早的优化是万恶之源。但是我们不应放弃那关键的 3% 优化机会。

--   Donald Knuth

即便是小的性能提升也能带来很大的改变，尤其是对于电子商务网站。现在留言本应用已经准备好迎接访问高峰，我们来看看如何检查它的性能。

找到性能优化点的最好途径就是使用一个 *分析器*。现在最流行的选择是 `Blackfire`_ （ *完全免责声明*：我也是 Blackfire 项目的创始人）。

Blackfire 介绍
----------------

Blackfire 由几部分组成：

* 一个用于触发分析的 *客户端程序* （可以是 Blackfire 命令行程序，也可以是针对谷歌或火狐浏览器的扩展）；

* 一个 *代理*，它会预备并聚合数据，然后发送给 blackfire.io 用于展示；

* 一个用于分析 PHP 代码的 PHP 扩展（即 *probe* 扩展）。

你要先在 Blackfire 上 `注册`_ 才能使用它。

运行下面这个快速安装脚本，它会在你的本地机器上安装 Blackfire：

.. code-block:: terminal
    :class: ignore

    $ curl https://installer.blackfire.io/installer.sh | bash

这个安装器会下载并安装 Blackfire 的命令行程序。

完成后，在所有可用的 PHP 版本里安装 PHP 的 probe 扩展：

.. code-block:: terminal
    :class: ignore

    $ sudo blackfire php:install

然后为我们的项目启用 PHP 的 probe 扩展：

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -7,3 +7,7 @@ session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
     xdebug.file_link_format=vscode://file/%f:%l
    +
    +[blackfire]
    +# use php_blackfire.dll on Windows
    +extension=blackfire.so

重启 web 服务器来让 PHP 加载 Blackfire：

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

Blackfire 命令行工具需要用你个人的 *客户端* 凭证来配置（这样你的项目配置才能存储在你的个人账号下）。你能在 ``Settings/Credentials`` `页`_ 的顶部找到这个凭证，用它来替换以下这个命令中的占位符，并且执行该命令：

.. code-block:: terminal
    :class: ignore

    $ blackfire client:config --client-id=xxx --client-token=xxx

在 Docker 上设置 Blackfire 代理
-------------------------------------

.. index::
    single: Docker;Blackfire
    single: Blackfire;Agent

Blackfire 代理服务已经在 Docker Compose 栈中配置好了：

.. code-block:: yaml
    :caption: compose.override.yaml
    :class: ignore

    ###> blackfireio/blackfire-symfony-meta ###
    blackfire:
        image: blackfire/blackfire:2
        # uncomment to store Blackfire credentials in a local .env.local file
        #env_file: .env.local
        environment:
        BLACKFIRE_LOG_LEVEL: 4
        ports: [8307]
    ###< blackfireio/blackfire-symfony-meta ###

你需要获取自己的 *服务器* 凭证才能和服务器通信（这些凭证会识别出你想要在哪里存储配置——你可以为每个项目创建一个凭证）；你可以在 ``Settings/Credentials`` `页`_ 底部找到它们。把它们存储为机密（secret）：

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

现在你可以启动新容器：

.. code-block:: terminal
    :class: ignore

    $ docker compose stop
    $ docker compose up -d --remove-orphans

针对 Blackfire 安装后无法运行进行修复
--------------------------------------------------

如果你在分析的时候出现错误，提高 Blackfire 日记级别，从而在日志中获得更多信息：

.. code-block:: diff
    :caption: patch_file
    :class: ignore

    --- i/php.ini
    +++ w/php.ini
    @@ -10,3 +10,4 @@ zend.detect_unicode=Off
     [blackfire]
     # use php_blackfire.dll on Windows
     extension=blackfire.so
    +blackfire.log_level=4

重启 web 服务器：

.. code-block:: terminal
    :class: ignore

    $ symfony server:stop
    $ symfony server:start -d

显示最近的日志：

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

再做一次分析并且检查日志输出。

在生产环境中配置 Blackfire
----------------------------------

.. index::
    single: Upsun;Blackfire

在所有的 Upsun 项目里，Blackfire 是默认包含的。

把 *服务器* 凭证设置为 **生产环境** 的机密：

.. code-block:: terminal
    :class: ignore

    $ symfony console secrets:set BLACKFIRE_SERVER_ID --env=prod
    # xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    $ symfony console secrets:set BLACKFIRE_SERVER_TOKEN --env=prod
    # xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

PHP 的 probe 扩展已经像其它需要的 PHP 扩展一样启用了：

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :emphasize-lines: 9
    :class: ignore

    runtime:
        extensions:
            - apcu
            - blackfire
            - ctype
            - iconv
            - mbstring
            - pdo_pgsql
            - sodium
            - xsl

为 Blackfire 配置 Varnish
----------------------------

.. index::
    single: Upsun;Varnish

你需要有一个方式绕过 Varnish 的 HTTP 缓存，才能部署并开始分析性能。不然的话，Blackfire 永远不会到达 PHP 应用程序。你只能授权来自于你本地机器的分析请求。

获取你当前的 IP 地址：

.. code-block:: terminal
    :class: ignore

    $ curl https://ifconfig.me/

用它来配置 Varnish：

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.vcl
    +++ w/.upsun/config.vcl
    @@ -1,3 +1,11 @@
    +acl profile {
    +   # Authorize the local IP address (replace with the IP found above)
    +   "192.168.0.1";
    +   # Authorize Blackfire servers
    +   "46.51.168.2";
    +   "54.75.240.245";
    +}
    +
     sub vcl_recv {
         set req.backend_hint = application.backend();
         set req.http.Surrogate-Capability = "abc=ESI/1.0";
    @@ -8,6 +16,16 @@ sub vcl_recv {
             }
             return (purge);
         }
    +
    +    # Don't profile ESI requests
    +    if (req.esi_level > 0) {
    +        unset req.http.X-Blackfire-Query;
    +    }
    +
    +    # Bypass Varnish when the profile request comes from a known IP
    +    if (req.http.X-Blackfire-Query && client.ip ~ profile) {
    +        return (pass);
    +    }
     }

     sub vcl_backend_response {

现在你可以部署了。

对网页做性能分析
------------------------

.. index::
    single: Profiling;Web Pages

你能通过火狐或谷歌浏览器的 `专用扩展`_ 来对传统的网页进行性能分析。

在本地电脑上分析性能时，别忘了在 ``config/packages/framework.yaml`` 文件中禁用 HTTP 缓存：不然的话，你分析的就是 Symfony 的 HTTP 缓存，而不是你自己的代码。

你需要对“生产”环境做性能分析，这样才能更好地了解生产环境中应用程序的性能如何。默认情况下，你的本地环境是“开发”环境，它会增加很多额外的性能开销（主要用来为 web 调试栏和 Symfony 分析器收集数据）。

.. note::

    由于我们将对“生产”环境进行性能分析，配置上无需改动什么，因为在前面一章里我们只为“开发”环境启用了 Symfony 的 HTTP 缓存层。

.. index::
    single: Symfony CLI;server:prod

通过修改 ``.env.local`` 文件里的 ``APP_ENV`` 环境变量，你把本地机器切换到了生产环境：

.. code-block:: text
    :class: ignore

    APP_ENV=prod

或者你可以用 ``server:prod`` 命令：

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod

当这次分析结束后，不要忘了切换回开发环境：

.. code-block:: terminal
    :class: ignore

    $ symfony server:prod --off

对 API 资源进行性能分析
--------------------------------

.. index::
    single: Profiling;API

对 API 进行分析的首选工具就是你之前安装的 Blackfire 命令行程序：

.. code-block:: terminal
    :class: ignore

    $ blackfire curl `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`api

``blackfire curl`` 命令接受的参数和选项与 `cURL`_ 完全一样。

对比性能
------------

在关于 “Cache” 的步骤中，我们为了提升代码性能增加了一个缓存层，但我们并没有去测量它带来的性能改变。由于我们都很难猜对什么会变快，什么会变慢，不排除最后你所做的优化反而会让应用变得更慢。

你应该总是要用分析器去衡量所做任何优化带来的影响。多亏有了 Blackfire 的 `比较功能`_，这种影响可以一目了然。

写黑盒功能测试
---------------------

.. index::
    single: Blackfire;Player

我们已经学了如何用 Symfony 写功能测试。可以用 Blackfire 写一些浏览情节，它们可以根据需要在 `Blackfire 播放器`_ 中运行。我们来写一个情节，它在开发环境中提交一条评论并且用邮件链接来验证，然后在生产环境中通过管理后台来验证。

用下面的内容创建 ``.blackfire.yaml`` 文件：

.. code-block:: text
    :caption: .blackfire.yaml

    scenarios: |
        #!blackfire-player

        group login
            visit url('/login')
            submit button("Sign in")
                param username "admin"
                param password "admin"
                expect status_code() == 302

        scenario
            name "Submit a comment on the Amsterdam conference page"
            include login
            visit url('/fr/conference/amsterdam-2019')
                expect status_code() == 200
            submit button("Submit")
                param comment[author] 'Fabien'
                param comment[email] 'me@example.com'
                param comment[text] 'Such a good conference!'
                param comment[photo] file(fake('simple_image', '/tmp', 400, 300, 'png', true, true), 'placeholder-image.jpg')
                expect status_code() == 302
            follow
                expect status_code() == 200
                expect not(body() matches "/Such a good conference/")
                # Wait for the workflow to validate the submissions
                wait 5000
            when env != "prod"
                visit url(webmail_url ~ '/messages')
                    expect status_code() == 200
                    set message_ids json("[*].id")
                with message_id in message_ids
                    visit url(webmail_url ~ '/messages/' ~ message_id ~ '.html')
                        expect status_code() == 200
                        set accept_url css("table a").first().attr("href")
                    include login
                    visit url(accept_url)
                        # we don't check the status code as we can deal
                        # with "old" messages which do not exist anymore
                        # in the DB (would be a 404 then)
            when env == "prod"
                visit url('/admin')
                    expect status_code() == 302
                follow
                click link("Comments")
                    expect status_code() == 200
                    set comment_ids css('table.table tbody tr').extract('data-id')
                with id in comment_ids
                    visit url('/admin/comment/review/' ~ id)
                        # we don't check the status code as we scan all comments,
                        # including the ones already reviewed
            visit url('/fr/')
                wait 5000
            visit url('/fr/conference/amsterdam-2019')
                expect body() matches "/Such a good conference/"

下载 Blackfire 播放器，这样才能在本地运行浏览情节：

.. code-block:: terminal

    $ curl -OLsS https://get.blackfire.io/blackfire-player.phar
    $ chmod +x blackfire-player.phar

在开发环境中运行浏览情节：

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL` .blackfire.yaml --variable "webmail_url=`symfony var:export MAILER_WEB_URL 2>/dev/null`" --variable="env=dev" -vv

.. code-block:: terminal
    :class: hide

    $ rm blackfire-player.phar

或是在生产环境中：

.. code-block:: terminal
    :class: ignore

    $ ./blackfire-player.phar run --endpoint=`symfony cloud:env:url --pipe --primary` .blackfire.yaml --variable "webmail_url=NONE" --variable="env=prod" -vv

通过增加 ``--blackfire`` 选项，Blackfire 的情节也可以为每个请求触发性能分析并且运行性能检测。

自动化性能检查
---------------------

管理性能不仅仅只是提高已有代码的性能，它也要确保代码的改动不会引起性能下降。

上节写的情节可用于在持续集成工作流中自动运行，也可以在生产环境中自动定期执行。

每次你在 Upsun 上创建新分支或是部署到生产环境来自动检测代码性能时，Upsun 也可以 `运行浏览情节`_。

.. sidebar:: 深入学习

    * `Blackfire 白皮书：PHP 代码性能详解`_；

    * `SymfonyCasts 的 Blackfire 教程`_。

.. _`Blackfire`: https://blackfire.io
.. _`注册`: https://blackfire.io/signup
.. _`页`: https://blackfire.io/my/settings/credentials
.. _`专用扩展`: https://blackfire.io/docs/integrations/browsers/index
.. _`cURL`: https://curl.haxx.se/docs/manpage.html
.. _`比较功能`: https://blackfire.io/docs/cookbooks/understanding-comparisons
.. _`Blackfire 播放器`: https://blackfire.io/player
.. _`运行浏览情节`: https://blackfire.io/docs/integrations/paas/platformsh#builds-level-enterprise
.. _`Blackfire 白皮书：PHP 代码性能详解`: https://blackfire.io/book
.. _`SymfonyCasts 的 Blackfire 教程`: https://symfonycasts.com/screencast/blackfire
