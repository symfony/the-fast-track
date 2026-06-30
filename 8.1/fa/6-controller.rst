ساخت یک کنترلر
==========================

.. index::
    single: Controller
    single: Routing;Route

پروژه‌ی guestbook در حال حاضر بر روی سرور عمل‌آوری در حال اجرا است، اما ما کمی تقلب کردیم. این پروژه هنوز هیچ صفحه‌ی وبی ندارد و صفحه‌ی اصلی، خطای کسل‌کننده‌ی 404 را نشان می‌دهد. بیایید درستش کنیم.

زمانی که یک درخواست HTTP می‌آید، همانند صفحه‌ی اصلی (``http://localhost:8000/``) ، سیمفونی تلاش می‌کند تا *راهی (route)* را که با *مسیر درخواستی (request path)* (دراینجا ``/``) تطابق دارد، پیدا کند. *راه (route)*، یک اتصال بین مسیر درخواستی و *PHP callable* است، یک تابع که *پاسخ* HTTP را برای آن درخواست می‌سازد.

این توابعِ قابل فراخوانی، «کنترلرها» نامیده می‌شوند.در سیمفونی، اکثر کنترلرها به صورت کلاس‌های PHP پیاده‌سازی می‌شوند. شما می‌توانید چنین کلاسی را به صورت دستی بسازید اما چون ما علاقه داریم که به سرعت پیش برویم، بیایید ببینیم که سمفونی چگونه می‌تواند به ما کمک کند.

تنبلی با باندلِ Maker
----------------------------------

.. index::
    single: Components;Maker Bundle
    single: Maker Bundle

برای تولید بدون زحمت کنترلرها، می‌توانیم از بسته‌ی ``symfony/maker-bundle`` استفاده کنیم که به عنوان بخشی از بسته‌ی ``webapp`` نصب شده است.

باندل maker در تولید کلاس‌های مختلف زیادی به شما کمک می‌کند. ما در این کتاب، تمام مدت از آن استفاده خواهیم کرد. هر "generator" در یک فرمان (command) تعریف می‌شود و تمام فرامین، بخشی از فضای نام (namespace) فرمانِ``make`` هستند.

.. index::
    single: Command;list

فرمان توکار ``list`` در کنسول سیمفونی، تمام فرامین موجود در یک  فضای نام را نمایش می‌دهد. از این فرمان برای کشف تمام generator‌های ارائه‌شده توسط باندل maker استفاده کنید:

.. code-block:: terminal
    :class: ignore

    $ symfony console list make

انتخاب یک قالب پیکربندی
-------------------------------------------

قبل از ایجاد اولین کنترلر پروژه، لازم است در مورد قالب‌های پیکربندی‌ای (configuration formats) که می‌خواهیم استفاده کنیم، تصمیم بگیریم. سیمفونی به صورت آماده از YAML، XML، PHP و attributeهای PHP پشتیبانی می‌کند.

برای *پیکربندی‌های مربوط به بسته‌ها*، *YAML* بهترین گزینه است. این قالب در پوشه‌ی ``config/`` استفاده شده است. اغلب هنگامی که یک بسته‌ی جدید نصب می‌کنید، recipe مربوط به آن بسته یک فایل جدید با پسوند ``.yaml`` به آن پوشه اضافه می‌کند.

برای *پیکربندی مربوط به کدهای PHP*، *attributeها* بهتر هستند زیرا در کنار کد تعریف می‌شوند. بگذارید با یک مثال توضیح دهم. وقتی یک درخواست می‌آید، تعدای پیکربندی لازم است تا به سیمفونی بگوید که مسیر درخواستی باید توسط یک کنترلر مشخص (یک کلاس PHP) رسیدگی شود. هنگامی که از قالب‌های YAML، XML یا PHP استفاده می‌کنید، دو فایل درگیر می‌شوند (فایل پیکربندی و فایل کنترلر PHP). زمانی که از attributeها استفاده می‌کنید، پیکربندی مستقیماً درون کلاس کنترلر انجام می‌شود.

شما ممکن است متعجب شوید که چگونه می‌توانید نام بسته‌ای که برای یک ویژگی لازم دارید را حدس بزنید؟ در اکثر اوقات لازم نیست که بدانید. در موارد زیادی، بسته‌ای که باید نصب شود، در پیغام‌های خطای سیمفونی وجود دارد. برای نمونه اجرای ``symfony console make:message`` بدون بسته‌ی ``messenger`` منجر به پیغام استثنایی شامل یک راهنمایی برای نصب بسته‌ی صحیح می‌گردد.

تولید یک کنترلر
----------------------------

.. index::
    single: Command;make:controller

اولین *کنترلر* خود را از طریق فرمان ``make:controller`` ایجاد کنید:

.. code-block:: terminal
    :class: answers(no)

    $ symfony console make:controller ConferenceController

.. index::
    single: Components;Routing
    single: Attributes;Route

این فرمان یک کلاس ``ConferenceController`` در داخل پوشه‌ی ``src/Controller/`` ایجاد می‌کند. کلاس تولید‌شده شامل مقداری کد الگو است که آماده‌اند تا با اصلاحات جزئی کارکرد مورد نظر را فراهم کنند:

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

attribute‌ی ``#[Route('/conference', name: 'app_conference')]`` چیزی است که باعث می‌شود متد ``index()`` یک کنترلر باشد (پیکربندی در کنار کدی است که پیکره‌بندی می‌شود).

زمانی که ``/conference`` را در مرورگر وارد کنید، کنترلر اجرا شده و پاسخ (response) بازگردانده می‌شود.

راه را تغییر دهید تا با صفحه‌ی اصلی مطابقت پیدا کند:

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

زمانی که می‌خواهیم به صفحه‌ی اصلی ارجاع دهیم، نامِ راه (route ``name``) می‌تواند مفید باشد. به جای هاردکد کردن مسیرِ ``/``، قصد داریم از نامِ راه استفاده کنیم.

بیایید به جای صفحه‌ی renderشده‌ی پیش‌فرض، یک HTML ساده بازگردانیم:

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

مرورگر را تازه‌سازی کنید:

.. figure:: screenshots/under-construction-homepage.png
    :alt: /
    :align: center
    :figclass: with-browser

وظیفه‌ی اصلی یک کنترلر، بازگرداندن یک پاسخ (HTTP ``Response``) برای درخواست است.

از آنجایی که باقی این فصل درباره‌ی کدی است که آن را نگه نخواهیم داشت، بیایید تغییراتمان را همین حالا commit کنیم:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Add the index controller'

.. _easter-egg:

اضافه کردن یک تخم‌مرغ عید پاک
------------------------------------------------------

برای اینکه نشان دهیم چگونه یک پاسخ می‌تواند از اطلاعات درخواست بهره بگیرد،  بیایید یک `Easter egg`_ کوچک اضافه کنیم. بیایید هر زمان که صفحه‌ی اصلی شامل یک رشته‌ی پرس‌وجو (query string) به صورت ``?hello=Fabien`` بود،  برای خوش‌آمدگویی به آن فرد، یک متن اضافه کنیم:

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

سیمفونی داده‌های درخواست را به صورت شیءِ ``Request`` ارائه می‌کند. هر زمان که سیمفونی یک آرگمانِ کنترلر (controller argument) با این type-hint ببیند، به صورت خودکار متوجه می‌شود که باید شیء درخواست را به آن بدهد. ما می‌توانیم از آن استفاده کنیم تا مقدار ``name`` را از رشته‌ی پرس‌وجو بگیریم و یک عنوان ``<h1>`` اضافه کنیم.

سعی کنید ابتدا ``/`` و سپس ``/?hello=Fabien`` را در مرورگر وارد کنید تا تفاوت را ببینید.

.. note::

    به فراخوانی ``htmlspecialchars()`` برای جلوگیری از مشکلات XSS دقت کنید. این چیزی است که وقتی از یک موتور قالب‌گیری مناسب استفاده کنیم (در بخش‌های آتی)، به صورت خودکار  برای ما انجام می‌شود.

ما همچنین می‌توانستیم نام را بخشی از URL قرار دهیم:

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

بخش ``{name}`` از راه، یک *پارامتر راه (route parameter)* پویا (dynamic) است که مشابه wildcard عمل می‌کند. حالا می‌توانید ابتدا ``/hello`` و سپس ``/hello/Fabien`` را در مرورگر وارد کنید تا همان نتایج قبلی را دریافت کنید. می‌توانید *مقدار* پارامتر ``{name}`` را با اضافه کردن یک آرگمان با نام یکسان *name* بگیرید. بنابراین، ``$name``.

تغییراتی که هم‌اکنون انجام دادیم را revert کنید:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

اشکال‌زدایی متغیرها
----------------------------

.. index::
    single: Components;VarDumper
    single: VarDumper
    single: dump

یک یاور فوق‌العاده در اشکال‌زدایی، تابع ``dump()`` در سیمفونی است. این تابع همواره در دسترس است و به شما این امکان را می‌دهد تا متغیر‌های پیچیده را به شکلی زیبا و تعاملی، دامپ (Dump) نمایید.

فایل ``src/Controller/ConferenceController.php`` را موقتاً تغییر دهید تا شیءِ «Request» را دامپ کند:

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

زمانی که صفحه را تازه‌سازی کردید، به آیکون جدید «target» در نوارابزار توجه کنید. این گزینه به شما اجازه می‌دهد تا دامپ را بررسی کنید. بر روی آن کلیک کنید تا به حالت تمام صفحه بروید و پیمایش راحت‌تر گردد:

.. figure:: screenshots/dumper.png
    :alt: /
    :align: center
    :figclass: with-browser

.. index::
    single: Git;checkout

تغییراتی که هم‌اکنون انجام دادیم را revert کنید:

.. code-block:: terminal

    $ git checkout src/Controller/ConferenceController.php

.. code-block:: terminal
    :class: hide

    $ git reset HEAD src/Controller/ConferenceController.php
    $ git checkout src/Controller/ConferenceController.php

.. sidebar:: بیشتر بدانید

    * سیستم `مسیریابی`_ سیمفونی؛

    * `آموزش تصویری راه‌ها، کنترلرها و صفحات در SymfonyCasts`_؛

    * `attributeهای PHP`_؛

    * `کامپوننتِ HttpFoundation`_؛

    * حمله‌های امنیتیِ `XSS (Cross-Site Scripting)`_؛

    * `برگه‌های تقلب مسیریابی سیمفونی`_.

.. _`Easter egg`: https://en.wikipedia.org/wiki/Easter_egg_(media)#In_computing
.. _`مسیریابی`: https://symfony.com/doc/current/routing.html
.. _`آموزش تصویری راه‌ها، کنترلرها و صفحات در SymfonyCasts`: https://symfonycasts.com/screencast/symfony/route-controller
.. _`attributeهای PHP`: https://www.php.net/attributes
.. _`کامپوننتِ HttpFoundation`: https://symfony.com/doc/current/components/http_foundation.html
.. _`XSS (Cross-Site Scripting)`: https://owasp.org/www-community/attacks/xss/
.. _`برگه‌های تقلب مسیریابی سیمفونی`: https://github.com/andreia/symfony-cheat-sheets/blob/master/Symfony4/routing_en_part1.pdf
