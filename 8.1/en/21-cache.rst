Caching for Performance
=======================

.. index::
    single: Components;HTTP Kernel
    single: HTTP Cache
    single: Cache

Performance problems might come with popularity. Some typical examples: missing database indexes or tons of SQL requests per page. You won't have any problems with an empty database, but with more traffic and growing data, it might arise at some point.

Adding HTTP Cache Headers
-------------------------

.. index::
    single: HTTP Cache;HTTP Cache Headers

Using HTTP caching strategies is a great way to maximize the performance for end users with little effort. Add a reverse proxy cache in production to enable caching, and use a `CDN`_ to cache on the edge for even better performance.

Let's cache the homepage for an hour:

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

The ``#[Cache]`` attribute configures the cache expiration for reverse proxies via its ``smaxage`` argument; use ``maxage`` to control the browser cache. Time is expressed in seconds (1 hour = 60 minutes = 3600 seconds). And like for routing or rate limiting, the caching policy is declared right where it applies: on the controller.

Caching the conference page is more challenging as it is more dynamic. Anyone can add a comment anytime, and nobody wants to wait for an hour to see it online. In such cases, use the *HTTP validation* strategy.

Activating the Symfony HTTP Cache Kernel
----------------------------------------

.. index::
    single: HTTP Cache;Symfony Reverse Proxy

To test the HTTP cache strategy, enable the Symfony HTTP reverse proxy, but only in the "development" environment (for the "production" environment, we will use a "more robust" solution):

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

Besides being a full-fledged HTTP reverse proxy, the Symfony HTTP reverse proxy (via the ``HttpCache`` class) adds some nice debug info as HTTP headers. That helps greatly in validating the cache headers we have set.

Check it on the homepage:

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

For the very first request, the cache server tells you that it was a ``miss`` and that it performed a ``store`` to cache the response. Check the ``cache-control`` header to see the configured cache strategy.

For subsequent requests, the response is cached (the ``age`` has also been updated):

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

Avoiding SQL Requests with ESI
------------------------------

.. index::
    single: HTTP Cache;ESI
    single: ESI

The ``TwigEventListener`` listener injects a global variable in Twig with all conference objects. It does so for every single page of the website. It is probably a great target for optimization.

You won't add new conferences every day, so the code is querying the exact same data from the database over and over again.

We might want to cache the conference names and slugs with the Symfony Cache, but whenever possible I like to rely on the HTTP caching infrastructure.

When you want to cache a fragment of a page, move it outside of the current HTTP request by creating a *sub-request*. *ESI* is a perfect match for this use case. An ESI is a way to embed the result of an HTTP request into another.

Create a controller that only returns the HTML fragment that displays the conferences:

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

Create the corresponding template:

.. code-block:: html+twig
    :caption: templates/conference/header.html.twig

    <ul>
        {% for conference in conferences %}
            <li><a href="{{ path('conference', { slug: conference.slug }) }}">{{ conference }}</a></li>
        {% endfor %}
    </ul>

Hit ``/conference_header`` to check that everything works fine.

.. index::
    single: Twig;render
    single: Twig;path

Time to reveal the trick! Update the Twig layout to call the controller we have just created:

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

And *voilà*. Refresh the page and the website is still displaying the same.

.. tip::

    Use the "Request / Response" Symfony profiler panel to learn more about the main request and its sub-requests.

Now, every time you hit a page in the browser, two HTTP requests are executed, one for the header and one for the main page. You have made performance worse. Congratulations!

The conference header HTTP call is currently done internally by Symfony, so no HTTP round-trip is involved. This also means that there is no way to benefit from HTTP cache headers.

Convert the call to a "real" HTTP one by using an ESI.

First, enable ESI support:

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

Then, use ``render_esi`` instead of ``render``:

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

If Symfony detects a reverse proxy that knows how to deal with ESIs, it enables support automatically (if not, it falls back to render the sub-request synchronously).

As the Symfony reverse proxy does support ESIs, let's check its logs (remove the cache first - see "Purging" below):

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

Refresh a few times: the ``/`` response is cached and the ``/conference_header`` one is not. We have achieved something great: having the whole page in the cache but still having one part dynamic.

This is not what we want though. Cache the header page for an hour, independently of everything else:

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

Cache is now enabled for both requests:

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

The ``x-symfony-cache`` header contains two elements: the main ``/`` request and a sub-request (the ``conference_header`` ESI). Both are in the cache (``fresh``).

The cache strategy can be different from the main page and its ESIs. If we have an "about" page, we might want to store it for a week in the cache, and still have the header be updated every hour.

Remove the listener as we don't need it anymore:

.. code-block:: terminal

    $ rm src/EventListener/TwigEventListener.php

Purging the HTTP Cache for Testing
----------------------------------

Testing the website in a browser or via automated tests becomes a little bit more difficult with a caching layer.

You can manually remove all the HTTP cache by removing the ``var/cache/dev/http_cache/`` directory:

.. code-block:: terminal

    $ rm -rf var/cache/dev/http_cache/

.. index::
    single: Attributes;Route

This strategy does not work well if you only want to invalidate some URLs or if you want to integrate cache invalidation in your functional tests. Let's add a small, admin only, HTTP endpoint to invalidate some URLs:

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

The new controller has been restricted to the ``PURGE`` HTTP method. This method is not in the HTTP standard, but it is widely used to invalidate caches.

By default, route parameters cannot contain ``/`` as it separates URL segments. You can override this restriction for the last route parameter, like ``uri``, by setting your own requirement pattern (``.*``).

The way we get the ``HttpCache`` instance can also look a bit strange; we are using an anonymous class as accessing the "real" one is not possible. The ``HttpCache`` instance wraps the real kernel, which is unaware of the cache layer as it should be.

Invalidate the homepage and the conference header via the following cURL calls:

.. code-block:: terminal

    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/
    $ curl -s -I -X PURGE -u admin:admin `symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`admin/http-cache/conference_header

The ``symfony var:export SYMFONY_PROJECT_DEFAULT_ROUTE_URL`` sub-command returns the current URL of the local web server.

.. note::

    The controller does not have a route name as it will never be referenced in the code.

Disabling the HTTP Cache in Development
---------------------------------------

The HTTP cache was great to validate our cache headers and to learn how to purge stale entries. But having a reverse proxy enabled in the development environment is unusual, and it quickly gets in the way: responses are served from the cache while you iterate on the code, and some vendor assets are even served with an empty body because of a long-standing HttpCache limitation with file responses.

Now that everything is validated, disable it; Varnish will take over in production:

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

Grouping similar Routes with a Prefix
-------------------------------------

.. index::
    single: Attributes;Route

The two routes in the admin controller have the same ``/admin`` prefix. Instead of repeating it on all routes, refactor the routes to configure the prefix on the class itself:

.. code-block:: diff
    :caption: patch_file

    --- i/src/Controller/AdminController.php
    +++ w/src/Controller/AdminController.php
    @@ -16,6 +16,7 @@ use Symfony\Component\Routing\Attribute\Route;
     use Symfony\Component\Workflow\WorkflowInterface;
     use Twig\Environment;

    +#[Route('/admin')]
     class AdminController extends AbstractController
     {
         public function __construct(
    @@ -25,7 +26,7 @@ class AdminController extends AbstractController
         ) {
         }

    -    #[Route('/admin/comment/review/{id}', name: 'review_comment')]
    +    #[Route('/comment/review/{id}', name: 'review_comment')]
         public function reviewComment(Request $request, Comment $comment, #[Target('comment')] WorkflowInterface $workflow): Response
         {
             $accepted = !$request->query->get('reject');
    @@ -51,7 +52,7 @@ class AdminController extends AbstractController
             ]));
         }

    -    #[Route('/admin/http-cache/{uri<.*>}', methods: ['PURGE'])]
    +    #[Route('/http-cache/{uri<.*>}', methods: ['PURGE'])]
         public function purgeHttpCache(KernelInterface $kernel, Request $request, string $uri, StoreInterface $store): Response
         {
             if ('prod' === $kernel->getEnvironment()) {

Caching CPU/Memory Intensive Operations
---------------------------------------

.. index::
    single: Process
    single: Components;Process

We don't have CPU or memory-intensive algorithms on the website. To talk about *local caches*, let's create a command that displays the current step we are working on (to be more precise, the Git tag name attached to the current Git commit).

The Symfony Process component allows you to run a command and get the result back (standard and error output).

Implement the command:

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

What if we want to cache the output for a few minutes? Use the Symfony Cache.

Symfony injects services type-hinted in the ``__invoke()`` method of a command, the same way it does for controller arguments. Wrap the code with the cache logic:

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

The process is now only called if the ``app.current_step`` item is not in the cache.

Profiling and Comparing Performance
-----------------------------------

Never add cache blindly. Keep in mind that adding some cache adds a layer of complexity. And as we are all very bad at guessing what will be fast and what is slow, you might end up in a situation where the cache makes your application slower.

Always measure the impact of adding a cache with a profiler tool like `Blackfire`_.

Refer to the step about "Performance" to learn more about how you can use Blackfire to test your code before deploying.

Configuring a Reverse Proxy Cache on Production
-----------------------------------------------

.. index::
    single: HTTP Cache;Varnish
    single: Upsun;Varnish
    single: Varnish

Instead of using the Symfony reverse proxy in production, we are going to use the "more robust" Varnish reverse proxy.

Add Varnish to the Upsun services:

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

Use Varnish as the main entry point in the routes:

.. code-block:: diff
    :caption: patch_file

    --- i/.upsun/config.yaml
    +++ w/.upsun/config.yaml
    @@ -1,5 +1,5 @@
     routes:
    -    "https://{all}/": { type: upstream, upstream: "app:http" }
    +    "https://{all}/": { type: upstream, upstream: "varnish:http", cache: { enabled: false } }
         "http://{all}/": { type: redirect, to: "https://{all}/" }

Finally, create a ``config.vcl`` file to configure Varnish:

.. code-block:: vcl
    :caption: .upsun/config.vcl

    sub vcl_recv {
        set req.backend_hint = application.backend();
    }

Enabling ESI Support on Varnish
-------------------------------

ESI support on Varnish should be enabled explicitly for each request. To make it universal, Symfony uses the standard ``Surrogate-Capability`` and ``Surrogate-Control`` headers to negotiate ESI support:

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

Purging the Varnish Cache
-------------------------

Invalidating the cache in production should probably never be needed, except for emergency purposes and maybe on non-``master`` branches. If you need to purge the cache often, it probably means that the caching strategy should be tweaked (by lowering the TTL or by using a validation strategy instead of an expiration one).

Anyway, let's see how to configure Varnish for cache invalidation:

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

In real life, you would probably restrict by IPs instead like described in the `Varnish docs`_.

Purge some URLs now:

.. code-block:: terminal

    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`
    $ curl -X PURGE -H 'x-purge-token: PURGE_NOW' `symfony cloud:env:url --pipe --primary`conference_header

The URLs looks a bit strange because the URLs returned by ``env:url`` already ends with ``/``.

.. sidebar:: Going Further

    * `Cloudflare`_, the global cloud platform;

    * `Varnish HTTP Cache docs`_;

    * `ESI specification`_ and `ESI developer resources`_;

    * `HTTP cache validation model`_;

    * `HTTP Cache in Upsun`_.

.. _`Blackfire`: https://blackfire.io/
.. _`Varnish docs`: https://varnish-cache.org/docs/trunk/users-guide/purging.html
.. _`CDN`: https://en.wikipedia.org/wiki/Content_delivery_network
.. _`Cloudflare`: https://www.cloudflare.com
.. _`Varnish HTTP Cache docs`: https://varnish-cache.org/docs/index.html
.. _`ESI specification`: https://www.w3.org/TR/esi-lang
.. _`ESI developer resources`: https://www.akamai.com/us/en/support/esi.jsp
.. _`HTTP cache validation model`: https://symfony.com/doc/current/http_cache/validation.html
.. _`HTTP Cache in Upsun`: https://symfony.com/doc/current/cloud/cookbooks/cache.html
