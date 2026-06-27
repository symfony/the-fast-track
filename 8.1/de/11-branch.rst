Den Code branchen
=================

Es gibt viele Möglichkeiten, den Workflow von Code-Änderungen in einem Projekt zu organisieren – Das direkte Arbeiten am Git Master-Branch und der direkte Einsatz im Produktivbetrieb ohne Tests ist aber nicht der beste Weg.

Beim Testen geht es nicht nur um Unit- oder Funktionale Tests, sondern auch um die Überprüfung des Anwendungsverhaltens mit echten Daten. Wenn Du oder Deine `Stakeholder`_ die Anwendung genau so benutzen können, wie sie für Endbenutzer*innen bereitgestellt wird, entsteht ein großer Vorteil, der Dir ermöglicht die Anwendung mit Vertrauen deployen zu können. Es ist besonders effizient, wenn nicht-technische Personen neue Funktionen validieren können.

Wir werden der Einfachheit halber (und zur Vermeidung von Wiederholungen) die nächsten Schritte auf dem Git Master-Branch fortführen, aber mal sehen, wie das besser funktionieren könnte.

Einen Git-Workflow einführen
-----------------------------

Ein möglicher Workflow ist die Erstellung eines Branches pro neuem Feature oder Bugfix. Das ist einfach und effizient.

Branches erstellen
------------------

.. index::
    single: Git;branch
    single: Git;checkout

Der Workflow beginnt mit der Erstellung eines Git-Branches:

.. code-block:: terminal
    :class: hide

    $ git branch -D sessions-in-db || true

.. code-block:: terminal

    $ git checkout -b sessions-in-db

Dieser Befehl erstellt einen ``sessions-in-db``-Branch, basierend auf dem ``master``-Branch. Es "spaltet" den Code und die Infrastrukturkonfiguration vom ``master``-Branch ab.

Sessions in der Datenbank speichern
-----------------------------------

.. index::
    single: Session;Database

Wie Du vielleicht schon am Namen des Branches erraten hast, wollen wir die Speicherung der Sessions vom Dateisystem auf (unsere PostgreSQL) Datenbank umstellen.

Die notwendigen Schritte, um dies zu verwirklichen, sind typisch:

#. Erstelle einen Git-Branch;

#. Aktualisiere bei Bedarf die Symfony-Konfiguration;

#. Schreibe und/oder aktualisiere bei Bedarf etwas Code;

#. Wenn nötig, aktualisiere die PHP-Konfiguration (füge zum Beispiel die PostgreSQL PHP-Erweiterung hinzu);

#. Aktualisiere die Infrastruktur auf Docker und Upsun falls nötig (füge den PostgreSQL-Dienst hinzu);

#. Teste lokal;

#. Teste remote;

#. Führe den Branch mit dem Master-Branch zusammen;

#. Deploye in die Produktivumgebung;

#. Lösche den Branch.

Um Sessions in der Datenbank zu speichern, ändere die ``session.handler_id``-Konfiguration so, dass sie auf den Datenbank-DSN weist:

.. code-block:: diff
    :caption: patch_file

    --- i/config/packages/framework.yaml
    +++ w/config/packages/framework.yaml
    @@ -3,7 +3,8 @@ framework:
         secret: '%env(APP_SECRET)%'

         # Note that the session will be started ONLY if you read or write from it.
    -    session: true
    +    session:
    +        handler_id: '%env(resolve:DATABASE_URL)%'

         #esi: true
         #fragments: true

Um Sessions in der Datenbank zu speichern, müüsen wir die ``sessions``-Tabelle anlegen. Mach das mit Doctrine Migrations:

.. code-block:: terminal

    $ symfony console make:migration

Führe die Datenbankmigration aus:

.. code-block:: terminal
    :class: answers(y)

    $ symfony console doctrine:migrations:migrate

Teste lokal, indem Du Dir die Website anschaust. Da es keine visuellen Veränderungen gibt und wir noch keine Sessions verwenden, sollte alles wie bisher funktionieren.

.. note::

    Wir brauchen Schritt 3 bis 5 hier nicht, weil wir die Datenbank erneut für die Session-Speicherung gebrauchen, aber das Kapitel wie man Redis gebraucht zeigt uns unkompliziert wie man einen neuen Dienst für Docker und Upsun hinzufügt, testet und bereitstellt.

Commit Deine Änderungen zu dem neuen Branch:

.. code-block:: terminal
    :class: ignore

    $ git add .
    $ git commit -m'Configure database sessions'

Einen Branch deployen
---------------------

.. index::
    single: Upsun;Environment

Bevor wir zum Produktivsystem deployen, sollten wir den Branch auf der gleichen Infrastruktur wie die Production-Environment testen. Wir sollten auch sicherstellen, dass für die Symfony ``prod``-Environment alles gut funktioniert (die lokale Website hat die Symfony ``dev``-Environment verwendet).

.. index::
    single: Symfony CLI;cloud:env:delete
    single: Symfony CLI;cloud:env:create

Lasst uns nun eine *Upsun-Environment* erstellen, die auf dem *Git-Branch* basiert:

.. code-block:: terminal
    :class: hide

    $ symfony cloud:env:delete sessions-in-db

.. code-block:: terminal

    $ symfony cloud:push

Dieser Befehl erstellt eine neue Environment:

* Der Branch erbt den Code und die Infrastruktur vom aktuellen Git-Branch (``sessions-in-db``);

* Die Daten stammen von der Master-Environment (auch bekannt als Production oder Produktivumgebung), und zwar durch eine Momentaufnahme aller Servicedaten, einschließlich Dateien (z. B. von Benutzer*innen hochgeladene Dateien) und Datenbanken;

* Ein neuer dedizierter Cluster wird erstellt, um den Code, die Daten und die Infrastruktur zu deployen.

Da das Deployment den gleichen Schritten folgt wie das Deployment in die Produktivumgebung, werden auch Datenbankmigrationen durchgeführt. Dies ist gleichzeitig eine gute Möglichkeit um sicherzugehen, dass die Migrationen mit echten Daten funktionieren.

Die Nicht-``master``-Environments sind der``master``-Environment sehr ähnlich, bis auf einige kleine Unterschiede: So werden beispielsweise E-Mails standardmäßig nicht gesendet.

.. index::
    single: Symfony CLI;cloud:url

Wenn das Deployment abgeschlossen ist, öffne den neuen Branch in einem Browser:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:url -1

Beachte, dass alle Upsun-Befehle mit dem aktuellen Git-Branch arbeiten. Somit wird der Befehl die URL für den ``sessions-in-db``-Branch aufrufen. Die URL sieht dann so ``https://sessions-in-db-xxx.eu-5.platformsh.site/`` aus.

Teste die Website auf dieser neuen Environment. Du solltest jetzt alle Daten sehen, die Du in der Master-Environment angelegt hast.

Wenn Du weitere Konferenzen in die ``master``-Environment hinzufügst, werden diese nicht in der ``sessions-in-db``-Environment angezeigt und umgekehrt. Die Environments sind unabhängig und isoliert.

Wenn sich der Code auf Master weiterentwickelt, kannst Du diese Änderungen jederzeit mittels ``rebase`` in den aktuellen Branch integrieren und die aktualisierte Version deployen, wodurch die Konflikte sowohl für den Code als auch für die Infrastruktur gelöst werden.

.. index::
    single: Symfony CLI;cloud:env:sync

Du kannst sogar die Daten von Master zurück in die ``sessions-in-db``-Environment synchronisieren:

.. code-block:: terminal
    :class: answers(y)

    $ symfony cloud:env:sync

Fehler von Deployments in die Produktivumgebung vermeiden
---------------------------------------------------------

.. index::
    single: Upsun;Debugging

Standardmäßig verwenden alle Upsun-Environments die Einstellungen der ``master``/``prod``-Environment (auch bekannt als die ``prod``-Symfony-Environment). Auf diese Weise kannst Du die Anwendung unter realen Bedingungen testen. Dies gibt Dir das Gefühl, direkt auf Produktivsystemen zu entwickeln und zu testen, aber ohne den damit verbundenen Risiken. Das erinnert mich an die guten alten Zeiten, als wir Deployments noch über FTP gemacht haben.

.. index::
    single: Symfony CLI;cloud:env:debug

Wenn ein Problem auftritt, möchtest Du vielleicht auf die ``dev``-Symfony-Environment wechseln:

.. code-block:: terminal

    $ symfony cloud:env:debug

Wenn Du fertig bist, gehe zurück zu den Produktiveinstellungen:

.. code-block:: terminal

    $ symfony cloud:env:debug --off

.. warning::

    Aktiviere **niemals** die ``dev``-Environment oder den Symfony Profiler im ``master``-Branch; dies würde Deine Anwendung wirklich langsam machen und viele ernsthafte Sicherheitsschwachstellen öffnen.

Produktivinstallationen vor dem Deployment testen
-------------------------------------------------

Der Zugriff auf die zukünftige Version der Website mit echten Daten eröffnet viele Möglichkeiten: vom visuellen Regressionstest bis zum Performance-Test. `Blackfire`_ ist das perfekte Werkzeug für diese Aufgabe.

Lies den Schritt über :doc:`Performance <29-performance>`, um mehr darüber zu erfahren, wie Du Blackfire verwenden kannst, um Deinen Code vor dem Deployment zu testen.

In die Produktivumgebung mergen
-------------------------------

.. index::
    single: Symfony CLI;deploy
    single: Git;checkout
    single: Git;merge

Wenn Du mit den Änderungen im Branch zufrieden bist, führe den Code und die Infrastruktur wieder in den Git Master-Branch zurück:

.. code-block:: terminal

    $ git checkout master
    $ git merge sessions-in-db

Und deploye:

.. code-block:: terminal

    $ symfony cloud:push

Beim Deployment werden nur die Code- und Infrastrukturänderungen zu Upsun übertragen; die Daten werden in keiner Weise beeinträchtigt.

Aufräumen
----------

.. index::
    single: Symfony CLI;env:delete
    single: Git;branch

Entferne zum Abschluss den Git-Branch und die Upsun-Environment:

.. code-block:: terminal

    $ git branch -d sessions-in-db
    $ symfony cloud:env:delete -e sessions-in-db

.. sidebar:: Weiterführendes

    * `Git Branching`_;

.. _`Stakeholder`: https://en.wikipedia.org/wiki/Project_stakeholder
.. _`Git Branching`: https://www.git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
.. _`Blackfire`: https://blackfire.io
