Solucionando problemas
======================

La creación de un proyecto también consiste en disponer de las herramientas adecuadas para depurar los problemas. Afortunadamente, muchos ayudantes interesantes ya están incluidos como parte del paquete ``webapp``.

Descubriendo las herramientas de depuración de Symfony
-------------------------------------------------------

.. index::
    single: Components;Profiler
    single: Profiler
    single: Web Profiler
    single: Web Debug Toolbar

Para empezar, Symfony Profiler es una herramienta que te ayuda a ahorrar tiempo cuando necesitas encontrar la causa raíz de un problema.

Si echas un vistazo a la página de inicio, deberías ver una barra de herramientas en la parte inferior de la pantalla:

.. figure:: screenshots/wdt.png
    :alt: /
    :align: center
    :figclass: with-browser

Lo primero que vas a notar es el **404** en rojo. Recuerda que es una página de muestra ya que aún no hemos definido una página de inicio. Incluso siendo hermosa la página por defecto que te da la bienvenida, sigue siendo una página de error. Así que el código de estado HTTP correcto es el 404, no el 200. Gracias a la barra de herramientas de depuración web, tienes la información de inmediato.

Si haces clic en el pequeño signo de exclamación, obtendrás el mensaje de excepción "real" almacenado como parte de los registros del *log* en Symfony Profiler. Si deseas ver la traza de la pila del error, haz clic en el enlace "Exception" en el menú de la izquierda.

Siempre que haya un problema con tu código, verás una página de excepción como la siguiente que te da todo lo que necesitas para entender el problema y de dónde viene:

.. figure:: screenshots/exception.png
    :alt: //
    :align: center
    :figclass: with-browser

Tómate tu tiempo para investigar la información que te ofrece Symfony Profiler haciendo clic sobre los distintos elementos.

.. index::
    single: Symfony CLI;server:log

Los registros del *log* también son muy útiles en las sesiones de depuración. Symfony tiene un comando útil para rastrear todos los registros del *log* (desde el servidor web, desde PHP y también desde tu aplicación):

.. code-block:: terminal
    :class: ignore

    $ symfony server:log

Vamos a hacer un pequeño experimento. Abre el fichero ``public/index.php`` y rompe su código PHP (añade foobar en el medio del código, por ejemplo). Actualiza la página en el navegador y observa el flujo del *log*:

.. code-block:: text
    :class: ignore

    Dec 21 10:04:59 |DEBUG| PHP    PHP Parse error:  syntax error, unexpected 'use' (T_USE) in public/index.php on line 5 path="/usr/bin/php7.42" php="7.42.0"
    Dec 21 10:04:59 |ERROR| SERVER GET  (500) / ip="127.0.0.1"

La salida está hermosamente coloreada para llamar tu atención sobre los errores.

Entendiendo los entornos de Symfony
-----------------------------------

.. index::
    single: Symfony Environments

Como Symfony Profiler sólo es útil durante la fase de desarrollo, queremos evitar que se instale en producción. De forma predeterminada, Symfony lo instaló automáticamente sólo para los entornos ``dev`` y ``test``.

Symfony soporta la noción de *entornos* . De forma predeterminada, tiene soporte integrado para tres entornos, pero puedes añadir tantos como desees: ``dev``, ``prod`` y ``test``. Todos los entornos comparten el mismo código, pero trabajan con *configuraciones* diferentes.

Por ejemplo, todas las herramientas de depuración están habilitadas en el entorno ``dev``. En el entorno ``prod`` la aplicación está optimizada para el rendimiento.

El cambio de un entorno a otro puede realizarse modificando la variable de entorno ``APP_ENV``.

Cuando desplegaste el proyecto en Upsun, el entorno (almacenado en ``APP_ENV``) se cambió automáticamente a ``prod``.

Gestionando la configuración de los entornos
---------------------------------------------

.. index::
    single: Environment Variables
    single: .env
    single: .env.local

La variable de entorno ``APP_ENV`` se puede establecer empleando variables de entorno "reales" en tu terminal:

.. code-block:: terminal
    :class: ignore

    $ export APP_ENV=dev

El uso de variables de entorno reales es la forma preferida de establecer valores como ``APP_ENV`` en los servidores de producción. Pero en las máquinas de desarrollo, tener que definir muchas variables de entorno puede ser engorroso. En su lugar, defínelos en un fichero ``.env``.

Un archivo ``.env`` con unos valores cuidadosamente elegidos se genera automáticamente cuando se crea el proyecto:

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

    Cualquier paquete puede añadir más variables de entorno a este archivo gracias a su receta utilizada por Symfony Flex.

El fichero ``.env`` se envía al repositorio y describe los valores *por defecto* del entorno de producción. Puede sustituir estos valores creando un fichero ``.env.local``. Este archivo no debe ser enviado al repositorio y es por ello que el fichero ``.gitignore`` ya lo está ignorando.

Nunca guardes datos secretos o confidenciales en estos archivos. Veremos cómo manejar los datos confidenciales en otro paso.

Configurando tu IDE
-------------------

En el entorno de desarrollo, cuando se lanza una excepción, Symfony muestra una página con el mensaje de excepción y el seguimiento de su pila. Cuando se muestra una ruta de archivo, añade un enlace que abre el archivo en la línea correspondiente en tu IDE favorito. Para beneficiarte de esta característica, necesitas configurar tu IDE. Symfony soporta muchos IDEs desde el primer momento; yo estoy usando Visual Studio Code para este proyecto:

.. code-block:: diff
    :caption: patch_file

    --- i/php.ini
    +++ w/php.ini
    @@ -6,3 +6,4 @@ session.gc_probability=0
     session.use_strict_mode=On
     realpath_cache_ttl=3600
     zend.detect_unicode=Off
    +xdebug.file_link_format=vscode://file/%f:%l

Los archivos vinculados no se limitan a las excepciones. Por ejemplo, el controlador en la barra de herramientas de depuración web permite hacer clic después de configurar el IDE.

Depurando en entorno de producción
-----------------------------------

.. index::
    single: Upsun;Remote Logs
    single: Upsun;SSH
    single: Symfony CLI;cloud:log
    single: Symfony CLI;cloud:ssh

La depuración de los entornos de producción siempre es más complicada. Por ejemplo, no tienes acceso a Symfony Profiler. Los registros son menos descriptivos. Pero es posible analizar estos registros del *log*:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:log --tail

Puedes incluso conectarte a través de SSH al contenedor web:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:ssh

No te preocupes, no puedes romper nada fácilmente. La mayor parte del sistema de ficheros es de sólo lectura. No podrás arreglar sobre la marcha en producción. Pero aprenderás una manera de solucionarlo mucho mejor más adelante en el libro.

.. sidebar:: Yendo más allá

    * `Tutorial de SymfonyCasts de entornos y archivos de configuración`_ ;

    * `Tutorial de SymfonyCasts de variables de entorno`_ ;

    * `Tutorial de SymfonyCasts de la barra de herramientas de depuración web y profiler`_ ;

    * `Gestión de múltiples archivos .env`_ en aplicaciones Symfony.

.. _`Tutorial de SymfonyCasts de entornos y archivos de configuración`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-config-files
.. _`Tutorial de SymfonyCasts de variables de entorno`: https://symfonycasts.com/screencast/symfony-fundamentals/environment-variables
.. _`Tutorial de SymfonyCasts de la barra de herramientas de depuración web y profiler`: https://symfonycasts.com/screencast/symfony/debug-toolbar-profiler
.. _`Gestión de múltiples archivos .env`: https://symfony.com/doc/current/configuration.html#managing-multiple-env-files
