Diagnostiquer les problèmes
============================

Mettre en place un projet, c'est aussi avoir les bons outils pour déboguer les problèmes. Fort heureusement, des assistants très utiles sont inclus avec le paquet ``webapp``.

Découvrir les outils de débogage de Symfony
---------------------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Pour commencer, le Symfony Profiler vous fait gagner un temps fou lorsque vous avez besoin de trouver l'origine d'un problème.

Si vous regardez la page d'accueil, vous devriez voir une barre d'outils en bas de l'écran :

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

La première chose que vous remarquerez, c'est le **404** en rouge. Rappelez-vous que ce n'est qu'une page de remplissage, car nous n'avons toujours pas défini de page d'accueil. Même si la page par défaut qui vous accueille est belle, c'est quand même une page d'erreur. Le code d'état HTTP correct est donc 404, pas 200. Grâce à la *web debug toolbar*, vous avez l'information tout de suite.

Si vous cliquez sur le petit point d'exclamation, vous obtenez le "vrai" message d'exception dans les logs du *Symfony Profiler*. Si vous voulez voir la *stack trace*, cliquez sur le lien "Exception" dans le menu de gauche.

Chaque fois qu'il y a un problème avec votre code, vous verrez une page d'exception comme celle-ci qui vous donnera tout ce dont vous aurez besoin pour comprendre le problème et d'où il vient :

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Prenez le temps d'explorer les informations à l'intérieur du profileur Symfony en cliquant partout.

.. index::
    single: Symfony CLI;server:log

Les logs sont également très utiles dans les sessions de débogage. Symfony a une commande pratique pour consulter tous les logs (du serveur web, de PHP et de votre application) :

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Réalisons une petite expérience. Ouvrez ``public/index.php`` et cassez le code PHP (ajoutez foobar au milieu du code par exemple). Rafraîchissez la page dans le navigateur et observez le contenu des logs :

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

Le résultat est joliment coloré pour attirer votre attention sur les erreurs.

Comprendre les environnements Symfony
-------------------------------------

.. index::
    single: Symfony Environments

Comme le Symfony Profiler n'est utile que pendant le développement, nous voulons éviter qu'il soit installé en production. Par défaut, Symfony ne l'installe que pour les environnements de ``dev`` et de ``test``.

Symfony intègre une notion d'*environnement*. Par défaut, il y en a trois, mais vous pouvez en ajouter autant que vous le souhaitez : ``dev``, ``prod`` et ``test``. Tous les environnements partagent le même code, mais ils représentent des *configurations* différentes.

Par exemple, tous les outils de débogage sont activés en environnement de ``dev``. Dans celui de ``prod``, l'application est optimisée pour la performance.

Basculer d'un environnement à l'autre peut se faire en changeant la variable d'environnement ``APP_ENV``.

Lorsque vous avez déployé vers Upsun, l'environnement (stocké dans ``APP_ENV``) a été automatiquement modifié en ``prod``.

Gérer la configuration des environnements
------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

``APP_ENV`` peut être défini en utilisant des variables d'environnement "réelles" depuis votre terminal :

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

L'utilisation de variables d'environnement réelles est la meilleure façon de définir des valeurs comme ``APP_ENV`` en production. Mais sur les machines de développement, avoir à définir beaucoup de variables d'environnement peut s'avérer fastidieux. Définissez-les plutôt dans un fichier ``.env``.

Un fichier sensible ``.env`` a été généré automatiquement pour vous lorsque le projet a été créé :

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

    N'importe quel paquet peut ajouter plus de variables d'environnement à ce fichier grâce à leur recette utilisée par Symfony Flex.

Le fichier ``.env`` est commité sur le dépôt Git et liste les valeurs *par défaut* de la production. Vous pouvez surcharger ces valeurs en créant un fichier ``.env.local``. Ce fichier ne doit pas être commité : c'est pourquoi le fichier ``.gitignore`` l'ignore déjà.

Ne stockez jamais des données secrètes ou sensibles dans ces fichiers. Nous verrons comment gérer ces données sensibles dans une autre étape.

Configurer votre IDE
--------------------

En environnement de développement, lorsqu'une exception est levée, Symfony affiche une page avec le message de l'exception et sa *stack trace*. Lors de l'affichage d'un chemin de fichier, un lien, qui ouvre le fichier à la bonne ligne dans votre IDE favori, est ajouté. Pour bénéficier de cette fonctionnalité, vous devez configurer votre IDE. Symfony supporte de nombreux IDE par défaut ; j'utilise Visual Studio Code pour ce projet :

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

Les fichiers liés ne sont pas limités à des exceptions. Par exemple, le contrôleur dans la *web debug toolbar* devient cliquable après avoir configuré l'IDE.

Déboguer en production
-----------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:log
    single: Symfony CLI;cloud:ssh

Le débogage des serveurs de production est toujours plus délicat. Vous n'avez pas accès au Symfony Profiler par exemple. Les logs sont moins détaillés. Mais il est possible de consulter les logs :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --tail

Vous pouvez même vous connecter en SSH sur le conteneur web :

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

Ne vous inquiétez pas, vous ne pouvez rien casser facilement. Une grande partie du système de fichiers est en lecture seule. Vous ne serez pas en mesure de faire un correctif urgent en production, mais vous apprendrez une manière bien plus adaptée de le faire plus loin dans le livre.

.. sidebar:: Aller plus loin

    * `SymfonyCasts : tutoriel sur les environnements et les fichiers de configuration`_ ;

    * `SymfonyCasts : tutoriel sur les variables d'environnement`_ ;

    * `SymfonyCasts : tutoriel sur la Web Debug Toolbar et le Profiler`_ ;

    * `Gérer plusieurs fichiers .env`_ dans les applications Symfony.

.. _`SymfonyCasts : tutoriel sur les environnements et les fichiers de configuration`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`SymfonyCasts : tutoriel sur les variables d'environnement`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`SymfonyCasts : tutoriel sur la Web Debug Toolbar et le Profiler`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`Gérer plusieurs fichiers .env`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
