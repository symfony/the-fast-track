Eine Datenbank einrichten
=========================

.. index::
    single: Database

Auf der Website des Konferenzgästebuchs geht es darum, während der Konferenzen Feedback zu sammeln. Wir müssen die Kommentare von den Konferenzteilnehmer*innen in einem permanenten Speicher ablegen.

Ein Kommentar lässt sich am besten durch eine feste Datenstruktur beschreiben: Autor*in, E-Mail, der Text des Feedbacks und ein optionales Foto. Das ist die Art Daten, die am besten in einer traditionellen relationalen Datenbank gespeichert wird.

PostgreSQL ist das Datenbanksystem unserer Wahl.

PostgreSQL zu Docker Compose hinzufügen
----------------------------------------

.. index::
    single: Docker;PostgreSQL

Auf unserem lokalen Rechner haben wir uns entschieden, Docker zur Verwaltung von Diensten zu verwenden. Die generierte ``compose.yml``-Datei beinhaltet bereits PostgreSQL als Dienst:

.. code-block:: yaml
    :caption: compose.yaml
    :emphasize-lines: 2,3
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        image: postgres:${POSTGRES_VERSION:-16}-alpine
        environment:
            POSTGRES_DB: ${POSTGRES_DB:-app}
            # You should definitely change the password in production
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ChangeMe}
            POSTGRES_USER: ${POSTGRES_USER:-app}
    volumes:
        - db-data:/var/lib/postgresql/data:rw
        # You may use a bind-mounted host directory instead, so that it is harder to accidentally remove the volume and lose all your data!
        # - ./docker/db/data:/var/lib/postgresql/data:rw
    ###< doctrine/doctrine-bundle ###

Dadurch wird ein PostgreSQL-Server installiert und einige Environment-Variablen konfiguriert, die den Datenbanknamen und die Credentials (Anmeldeinformationen) steuern. Die Werte spielen keine Rolle.

Wir stellen auch den PostgreSQL-Port (``5432``) des Containers dem lokalen Host zur Verfügung. Das wird uns helfen, von unserer Maschine aus auf die Datenbank zuzugreifen:

.. code-block:: yaml
    :caption: compose.override.yaml
    :emphasize-lines: 4
    :class: ignore

    ###> doctrine/doctrine-bundle ###
    database:
        ports:
        - "5432"
    ###< doctrine/doctrine-bundle ###

.. note::

    Die ``pdo_pgsql``-Erweiterung sollte installiert worden sein, als PHP in einem vorherigen Schritt eingerichtet wurde.

Docker Compose starten
----------------------

Starte Docker Compose im Hintergrund (``-d``):

.. code-block:: terminal
    :class: hide

    $ docker compose down --remove-orphans

.. code-block:: terminal

    $ docker compose up -d --remove-orphans

Warte ein wenig, bis die Datenbank hochgefahren ist und überprüfe, ob alles in Ordnung ist:

.. code-block:: terminal
    :class: ignore

    $ docker compose ps

            Name                      Command              State            Ports
    ---------------------------------------------------------------------------------------
    guestbook_database_1   docker-entrypoint.sh postgres   Up      0.0.0.0:32780->5432/tcp

Überprüfe die Docker Compose logs wenn es keine aktiven Container gibt oder wenn die ``State``-Spalte nicht ``Up`` anzeigt:

.. code-block:: terminal
    :class: ignore

    $ docker compose logs

Zugriff auf die lokale Datenbank
--------------------------------

Die Nutzung des ``psql``-Kommandozeilenprogramms kann manchmal hilfreich sein. Aber Du musst dir Zugangsdaten und den Datenbanknamen merken. Weniger offensichtlich ist, dass Du auch den lokalen Port kennen musst, auf dem die Datenbank auf dem Host läuft. Docker wählt einen zufälligen Port, damit Du gleichzeitig an mehreren Projekten mit PostgreSQL arbeiten kannst (der lokale Port ist Teil der Ausgabe von ``docker compose ps``).

Wenn Du ``psql`` über die Symfony CLI  aufrufst, musst Du dir nichts merken.

Die Symfony CLI erkennt automatisch die für das Projekt ausgeführten Docker-Dienste und stellt die Environment-Variablen bereit, die ``psql`` für die Verbindung zur Datenbank benötigt.

.. index::
    single: Symfony CLI;run psql

Dank dieser Konventionen ist der Zugriff auf die Datenbank via ``symfony run`` viel einfacher:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

.. note::

    Wenn Du die ``psql`` Binärdatei nicht auf Deinem lokalen Host hast, kannst Du sie auch über ``docker compose`` laufen lassen:

    .. code-block:: terminal
        :class: ignore

        $ docker compose exec database psql app app

Datenbank-Daten exportieren und importieren
-------------------------------------------

.. index::
    single: Database;Dump
    single: Symfony CLI;run pg_dump
    single: Symfony CLI;run psql

Verwende ``pg_dump`` um die Datenbank-Daten zu exportieren:

.. code-block:: terminal
    :class: ignore

    $ symfony run pg_dump --data-only > dump.sql

Und importiere die Daten mit:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql < dump.sql

PostgreSQL zu Upsun hinzufügen
-------------------------------------

.. index::
    single: Upsun;PostgreSQL

Für die Produktiv-Infrastruktur auf Upsun sollte das Hinzufügen eines Dienstes wie PostgreSQL in der ``.upsun/config.yaml``-Datei erfolgen, das wurde bereits durch das Recipe vom ``webapp``-Paket gemacht:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    database:
        type: postgresql:16

Der ``database``-Dienst ist eine PostgreSQL-Datenbank (in der selben Version wie für Docker). Upsun weist ihren Speicher beim ersten Deployment automatisch zu; passe ihn bei Bedarf später mit ``symfony cloud:resources:set`` an.

Wir müssen die Datenbank auch mit dem Anwendungscontainer "verknüpfen", was in ``.upsun/config.yaml`` beschrieben ist:

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    relationships:
        database: "database:postgresql"

Der ``database``-Dienst vom Typ ``postgresql`` wird  auf dem Anwendungscontainer mit ``database`` referenziert.

Kontrolliere, daß die ``pdo_pgsql``-Erweiterung bereits installiert ist für die PHP-Runtime.

.. code-block:: yaml
    :caption: .upsun/config.yaml
    :class: ignore

    runtime:
        extensions:
            # other extensions
            - pdo_pgsql
            # other extensions

Zugriff auf die Upsun-Datenbank
-------------------------------------

PostgreSQL läuft nun sowohl lokal über Docker, als auch in der Produktivumgebung auf Upsun.

Wie wir gerade gesehen haben, verbindet ``symfony run psql`` sich automatisch mit der von Docker gehosteten Datenbank – Dank der Environment-Variablen, die von ``symfony run`` bereitgestellt werden.

.. index::
    single: Upsun;Tunnel
    single: Symfony CLI;cloud:tunnel:open
    single: Symfony CLI;cloud:tunnel:close
    single: Symfony CLI;var:expose-from-tunnel
    single: Symfony CLI;run psql

Wenn Du eine Verbindung zur PostgreSQL-Datenbank herstellen möchtest, die auf den Production-Containern gehostet wird, kannst Du einen SSH-Tunnel zwischen dem lokalen Computer und der Upsun-Infrastruktur öffnen:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel

Standardmäßig werden Upsun-Dienste nicht als Environment-Variablen auf dem lokalen Rechner angezeigt. Dafür musst Du zusätzlich den ``var:expose-from-tunnel``-Befehl ausführen. Warum? Die Verbindung zur Datenbank in der Produktivumgebung ist ein gefährlicher Vorgang. Du kannst mit *echten* Daten herumpfuschen.

Verbinde dich nun via ``symfony run psql`` wie bisher mit der remote PostgreSQL-Datenbank:

.. code-block:: terminal
    :class: ignore

    $ symfony run psql

Wenn du fertig bist, vergiss nicht, den Tunnel zu schließen:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:close

.. tip::

    Um einige SQL-Abfragen auf der Production-Datenbank auszuführen, kannst Du auch den ``symfony sql``-Befehl anstelle einer Shell verwenden.

Environment-Variablen bereitstellen
-----------------------------------

.. index::
    single: Upsun;Environment Variables
    single: Symfony CLI;var:export

Docker Compose und Upsun arbeiten dank Environment-Variablen nahtlos mit Symfony zusammen.

Überprüfe alle Environment-Variablen, die durch ``symfony`` bereitgestellt werden, indem Du ``symfony var:export`` ausführst:

.. code-block:: terminal
    :class: ignore

    $ symfony var:export

    PGHOST=127.0.0.1
    PGPORT=32781
    PGDATABASE=app
    PGUSER=app
    PGPASSWORD=!ChangeMe!
    # ...

Die ``PG*`` Environment-Variablen werden vom ``psql`` Dienstprogramm gelesen. Was ist mit den anderen?

Wenn ein Tunnel zu Upsun mit ``var:expose-from-tunnel`` geöffnet ist, gibt der ``var:export``-Befehl die Environment-Variablen in der Upsun zurück:

.. code-block:: terminal
    :class: ignore

    $ symfony cloud:tunnel:open
    $ symfony var:expose-from-tunnel
    $ symfony var:export
    $ symfony cloud:tunnel:close

Deine Infrastruktur beschreiben
-------------------------------

Du hast es vielleicht noch nicht gemerkt, aber es ist sehr hilfreich die Infrastruktur mit beim Code zu speichern. Docker und Upsun nutzen Konfigurationsdateien um die Infrastruktur des Projektes zu beschreiben. Wenn eine neue Funktionalität einen zusätzlichen Service braucht, sind die Änderungen des Code und die Änderungen für die Infrastruktur im selben Patch.

.. sidebar:: Weiterführendes

    * `Upsun-Dienste`_;

    * `Upsun-Tunnel`_;

    * `PostgreSQL-Dokumentation`_;

    * die `Docker-Compose-Befehle`_.

.. _`Upsun-Dienste`: https://symfony.com/doc/current/cloud/services/intro.html#available-services
.. _`Upsun-Tunnel`: https://symfony.com/doc/current/cloud/services/intro.html#connecting-to-a-service
.. _`PostgreSQL-Dokumentation`: https://www.postgresql.org/docs/
.. _`Docker-Compose-Befehle`: https://docs.docker.com/reference/cli/docker/compose/
