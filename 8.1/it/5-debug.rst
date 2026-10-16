Risoluzione dei problemi
========================

Impostare un progetto significa, tra le altre cose, avere gli strumenti corretti per il debug dei problemi. Fortunatamente tanti di questi sono inclusi nel pacchetto ``webapp``.

Alla scoperta degli strumenti di debug di Symfony
-------------------------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Per cominciare, il Profiler di Symfony ci aiuterà a risparmiare un sacco di tempo quando avremo bisogno di cercare la causa di un problema.

Dando un'occhiata alla homepage, dovremmo vedere una barra degli strumenti nella parte inferiore dello schermo:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

La prima cosa che possiamo notare è il **404** in rosso. Ricordiamoci che questa pagina è una pagina di cortesia poiché non abbiamo ancora definito una homepage. Anche se la pagina di default che vi da il benvenuto è bella, rimane sempre una pagina di errore, quindi il codice di status HTTP è 404 e non 200. Grazie alla barra degli strumenti di debug abbiamo immediatamente questa informazione.

Cliccando sul piccolo punto esclamativo, si potrà leggere il "vero" messaggio d'eccezione facente parte dei log nel Profiler di Symfony. Se volessimo vedere lo stack trace, potremmo cliccare sul link "Exception" nel menù di sinistra.

Ogni volta che ci sarà un problema con il codice, vedremo una pagina d'eccezione come la seguente. Ci darà tutto il necessario per capire il problema e da dove provenga:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Prendetevi un po' di tempo per esplorare le informazioni all'interno del Profiler di Symfony, cliccandoci sopra.

.. index::
    single: Symfony CLI;server:log

I log sono anche molto utili durante il debug. Symfony ha un comodo comando per il tail di tutti i log (dal server web, da PHP e dall'applicazione):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Facciamo un piccolo esperimento aprendo il file ``public/index.php`` e rompendo il codice PHP (ad esempio aggiungendo la stringa "foobar" in mezzo al codice). Aggiorniamo la pagina web nel browser e osserviamo i log:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

L'output è colorato per attirare l'attenzione sugli errori.

Comprendere gli ambienti di Symfony
-----------------------------------

.. index::
    single: Symfony Environments

Siccome il Symfony Profiler è utile solo in fase di sviluppo, vogliamo evitare che venga installato in produzione. Di base, Symfony lo installa solo per gli ambienti di ``dev`` e ``test``.

Symfony supporta il concetto di *ambienti*. Per impostazione predefinita il supporto integrato ne prevede tre: ``dev``, ``prod`` e ``test``. È comunque possibile aggiungerne quanti ne vogliamo. Tutti gli ambienti condividono lo stesso codice, ma rappresentano *configurazioni* differenti.

Per esempio, tutti gli strumenti di debug sono abilitati nell'ambiente ``dev``. Nell'ambiente ``prod``, l'applicazione è ottimizzata per le prestazioni.

Il passaggio da un ambiente all'altro può essere fatto cambiando la variabile d'ambiente ``APP_ENV``

Quando abbiamo eseguito il deploy su Upsun, l'ambiente (memorizzato in ``APP_ENV``) è stato automaticamente cambiato in ``prod``.

Gestire le configurazioni d'ambiente
------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` può essere impostata utilizzando variabili d'ambiente "reali" nel terminale:

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

Utilizzare variabili di ambiente reali è il modo migliore per impostare i valori come ``APP_ENV`` sui server di produzione. Dover definire molte variabili d'ambiente sulla macchina di sviluppo, però, può essere poco pratico. Le possiamo dunque definire in un file ``.env``.

Un file ``.env`` è stato generato automaticamente per noi quando il progetto è stato creato:

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

    Grazie alle ricette usate da Symfony Flex, qualsiasi pacchetto può aggiungere altre variabili d'ambiente a questo file.

Facciamo commit del file ``.env``, che descrive i valori *predefiniti* per l'ambiente di produzione. Possiamo sovrascrivere questi valori creando un file ``.env.local``. Di quest'ultimo file va evitato il commit ed è per questo che ``.gitignore`` lo sta già escludendo.

Non memorizziamo mai valori segreti o sensibili in questi file. Vedremo più avanti come gestirli.

Configurazione dell'IDE
-----------------------

Nell'ambiente di sviluppo, quando viene lanciata un'eccezione, Symfony mostra una pagina con il messaggio di eccezione e il suo stack trace. Quando si visualizza il percorso di un file, aggiunge un link che apre il file nel nostro IDE preferito, portandoci direttamente alla riga corrispettiva al link cliccato. Per poter utilizzare questa funzione è necessario configurare l'IDE. Symfony supporta molti IDE già pronti all'uso; io sto usando Visual Studio Code per questo progetto:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

I file collegati non sono limitati alle eccezioni. Per esempio, il controller nella barra degli strumenti di debug diventa cliccabile dopo aver configurato l'IDE.

Debug in produzione
-------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:log
    single: Symfony CLI;cloud:ssh

Il debug sui server di produzione è spesso più complicato. Non si ha accesso al Profiler di Symfony e i log sono meno verbosi, ma è possibile farne un tail:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --tail

È anche possibile connettersi al container web via SSH:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

Nessuna paura, non potremo rompere nulla così facilmente. La maggior parte del filesystem è in sola lettura quindi non sarà possibile fare "hot fix" in produzione, vedremo un modo migliore più avanti.

.. sidebar:: Andare oltre

    * `Tutorial su ambienti e file di configurazione in SymfonyCasts`_;

    * `Tutorial sulle variabili d'ambiente in SymfonyCasts`_;

    * `Tutorial su barra degli strumenti di debug e Profiler in SymfonyCasts`_;

    * `Gestire più file .env`_ nelle applicazioni Symfony.

.. _`Tutorial su ambienti e file di configurazione in SymfonyCasts`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`Tutorial sulle variabili d'ambiente in SymfonyCasts`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`Tutorial su barra degli strumenti di debug e Profiler in SymfonyCasts`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`Gestire più file .env`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
