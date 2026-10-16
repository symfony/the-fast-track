Rozwiązywanie problemów
=========================

Konfiguracja projektu polega również na posiadaniu odpowiednich narzędzi do debugowania. Na szczęście wiele przydatnych narzędzi jest już zawartych w pakiecie ``webapp``.

Odkrywanie narzędzi Symfony do debugowania
-------------------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Na początek, Symfony Profiler, który oszczędza czas podczas szukania źródła problemu.

Jeśli spojrzysz na stronę główną, u dołu ekranu powinieneś/aś zobaczyć pasek narzędzi:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Pierwszą rzeczą, jaką możesz zauważyć, jest czerwone **404**. Pamiętaj, że ta strona jest elementem tymczasowym, ponieważ nie zdefiniowaliśmy jeszcze strony głównej. Nawet jeśli strona domyślna, która Cię wita, jest całkiem ładna, to jest to tylko strona błędu. Więc prawidłowy kod statusu HTTP to 404, a nie 200. Dzięki paskowi narzędzi do debugowania widzisz to od razu.

Jeśli klikniesz na mały wykrzyknik, otrzymasz pełną informację opisującą wyjątek jako część logów w profilerze Symfony. Jeśli chcesz zobaczyć ślad stosu (ang. stack trace), kliknij na odnośnik "Exception" w lewym menu.

W przypadku pojawienia się problemu z kodem zobaczysz stronę z wyjątkiem, taką jak poniższa, która daje Ci wszystko, czego potrzebujesz, aby zrozumieć problem i źródło jego pochodzenia:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Poświęć trochę czasu na poznanie informacji zawartych w profilerze Symfony klikając na niego.

.. index::
    single: Symfony CLI;server:log

Logi są również bardzo przydatne podczas debugowania. Symfony posiada wygodne polecenie śledzenia wszystkich logów (z serwera WWW, PHP i Twojej aplikacji):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Przeprowadźmy mały eksperyment. Otwórz ``public/index.php`` i zepsuj coś w kodzie PHP (np. dodaj foobar w środku kodu). Odśwież stronę w przeglądarce i obserwuj strumień dziennika:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

Wyjście jest pięknie pokolorowane, aby zwrócić uwagę na błędy.

Zrozumienie środowisk Symfony
------------------------------

.. index::
    single: Symfony Environments

Ponieważ Symfony Profiler jest przydatny tylko podczas developmentu, chcemy uniknąć instalowania go w środowisku produkcyjnym. Domyślnie Symfony instaluje go tylko dla środowisk ``dev`` i ``test``.

Symfony w pełni współgra z ideą *środowisk* w aplikacji. Framework ten wspiera domyślnie trzy różne środowiska: ``dev``, ``prod`` i ``test``, ale możesz dodać kolejne. Wszystkie środowiska współdzielą ten sam kod, ale wykorzystują różne *konfiguracje*.

Na przykład wszystkie narzędzia do debugowania są włączone w środowisku ``dev``. W środowisku ``prod`` aplikacja jest zoptymalizowana pod kątem wydajności.

Przełączanie się z jednego środowiska do drugiego można wykonać poprzez zmianę zmiennej środowiskowej ``APP_ENV``.

Po wdrożeniu na platformę Upsun, środowisko (przechowywane w ``APP_ENV``) jest automatycznie przełączane na ``prod``.

Zarządzanie konfiguracjami środowisk
--------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

Zmienna ``APP_ENV`` może być ustawiona za pomocą "prawdziwych" zmiennych środowiskowych w terminalu:

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

Korzystanie z rzeczywistych zmiennych środowiskowych jest rekomendowanym sposobem ustawiania wartości takich jak ``APP_ENV`` na serwerach produkcyjnych. Ale na maszynach deweloperskich konieczność zdefiniowania wielu zmiennych środowiskowych może być uciążliwa. Zamiast tego definiuj je w pliku ``.env``.

Gotowy plik ``.env`` został wygenerowany automatycznie w momencie tworzenia projektu:

.. code-block:: text
    :caption: .env
    :class: ignore

    ###> symfony/framework-bundle ###
    APP_ENV=dev
    APP_SECRET=c2927f273163f7225a358e3a1bbbed8a
    #TRUSTED_PROXIES=127.0.0.1,127.0.0.2
    #TRUSTED_HOSTS='^localhost|example\.com$'
    ###< symfony/framework-bundle ###

.. tip::

    Każdy pakiet może dodać więcej zmiennych środowiskowych do tego pliku dzięki przepisowi (ang. recipe) używanemu przez Symfony Flex.

Plik ``.env``, który zawiera wartości *domyślne* środowiska produkcyjnego, jest przechowywany w repozytorium. Możesz nadpisać te wartości tworząc plik ``.env.local``. Plik ten nie powinien być zatwierdzany (ang. commit) i dlatego ``.gitignore`` domyślnie go ignoruje.

Nigdy nie przechowuj poufnych lub wrażliwych danych w tych plikach. W kolejnym kroku zobaczymy, jak zarządzać poufnymi danymi.

Skonfiguruj swoje IDE
---------------------

W środowisku deweloperskim, kiedy wyjątek zostaje rzucony, Symfony wyświetla stronę z komunikatem o wyjątku i jego stosem. Podczas wyświetlania ścieżki do pliku dodaje odnośnik, który otwiera plik we właściwej linii w Twoim ulubionym IDE, jeśli je odpowiednio skonfigurujesz. Symfony obsługuje wiele IDE tuż po instalacji - w tym projekcie używam Visual Studio Code:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

Odnośniki do plików nie są ograniczone tylko do wyjątków. Dla przykładu, kontroler w pasku narzędzi do debugowania staje się klikalny po odpowiednim skonfigurowaniu IDE.

Debugowanie w środowisku produkcyjnym
--------------------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:log
    single: Symfony CLI;cloud:ssh

Debugowanie serwerów produkcyjnych jest zawsze trudniejsze. Nie masz dostępu na przykład do profilera Symfony. Dzienniki logów są mniej szczegółowe, ale śledzenie ostatnich zapisów jest możliwe:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --tail

Możesz nawet połączyć się przez SSH z kontenerem webowym:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

Nie martw się, niczego łatwo nie zepsujesz. Większa część systemu plików jest tylko do odczytu. Nie będziesz w stanie wykonać pilnej poprawki w środowisku produkcyjnym, ale nie przejmuj się, w kolejnych rozdziałach tej książki nauczysz się o wiele lepszego podejścia.

.. sidebar:: Idąc dalej

    * `SymfonyCasts: przewodnik po środowiskach i plikach konfiguracyjnych`_;

    * `SymfonyCasts: przewodnik po zmiennych środowiskowych`_;

    * `SymfonyCasts: przewodnik po pasku narzędzi do debugowania i profilerze`_;

    * `Zarządzanie wieloma plikami .env`_ w aplikacjach Symfony.

.. _`SymfonyCasts: przewodnik po środowiskach i plikach konfiguracyjnych`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`SymfonyCasts: przewodnik po zmiennych środowiskowych`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`SymfonyCasts: przewodnik po pasku narzędzi do debugowania i profilerze`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`Zarządzanie wieloma plikami .env`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
