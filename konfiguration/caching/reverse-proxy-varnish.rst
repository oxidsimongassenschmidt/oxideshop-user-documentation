﻿Reverse Proxy Varnish
=====================

Funktionsweise
--------------
Varnish ist ein Reverse Proxy, der vor dem eigentlichen Webserver eingehende Anfragen von Web-Clients verarbeitet. Die Webseiten, welche an die Web-Clients ausgeliefert werden, werden zum größten Teil aus zwischengespeicherten Inhalten zusammengestellt. Erst wenn die Lebensdauer des Caches abgelaufen ist, fragt Varnish Inhalte vom Webserver ab. Der OXID eShop muss dann die angeforderten Daten aus der Datenbank lesen und bereitstellen.

Die Verarbeitung durch Varnish basiert auf der Aufteilung der einzelnen Seiten des OXID eShop in kleine Teilbereiche, so genannte Widgets. Für Varnish werden diese Widgets als ESI-Tags gekennzeichnet. Dadurch kann Varnish dynamische Seitenbereiche, wie beispielsweise Warenkorb oder Login, separat abfragen und aktualisieren. Darüber hinaus werden Grafiken, Artikel- und Kategoriebilder, Stylesheet- und JavaScript-Dateien immer zwischengespeichert.

.. image:: ../../media/screenshots/oxbacb01.png
   :alt: Web-Clients, Varnish und Server mit OXID eShop
   :height: 204
   :width: 650

Um den Reverse Proxy Varnish für das Caching des OXID eShop nutzen zu können, muss Varnish auf einem separaten Server installiert und konfiguriert werden.

-------------------------------------------------

Installation
------------
Installieren Sie Varnish auf einem Server. Die Software und die Anleitung zur Installation erhalten Sie auf der Website des Herstellers: `http://www.varnish-cache.org <http://www.varnish-cache.org/>`_ .

-------------------------------------------------

Konfiguration
-------------
Varnish verfügt über eine eigene Sprache, mit welcher dessen Verhalten konfiguriert werden kann. Mit VCL (Varnish Configuration Language) definiert, wird die Konfiguration in Binärcode übersetzt und bei Anfragen aus dem Web ausgeführt. Die Standard-Konfigurationsdatei ist :file:`default.vcl` und befindet sich im Verzeichnis :file:`/etc/varnish`.

Wir haben die Konfigurationsdatei :file:`default.vcl` für das Caching des OXID eShop angepasst. Die Definitionen entsprechen einem Standard-Shop. Sie sollten nur bei einem stark angepassten Shop verändert werden. Das betrifft ausschließlich Veränderungen im Standardverhalten des Shops durch modulare Erweiterungen, beispielsweise komplett geänderter Umgang mit Artikeln oder Einsatz eigener Cookies. Die notwendigen Änderungen in der Konfigurationsdatei setzen profunde Kenntnisse der VCL voraus. Fehlerhafte Anweisungen können die Performance beeinträchtigen und den Shop Daten bereitstellen lassen, die nicht aktuell sind. Wird die Konfigurationsdatei unverändert in einem stark angepassten Shop verwendet, kann das zu Datenverlust und unerwartetem Verhalten führen.

Die Konfigurationsdatei :file:`servers_conf.vcl` enthält Hostnamen und IPs der beteiligten Server und muss an die reale Systemumgebung angepasst werden.

Varnish ab Version 4.0.3
^^^^^^^^^^^^^^^^^^^^^^^^
Die ausgelieferte Datei :file:`default.vcl` enthält die Konfiguration für Varnish ab Version 4.0.3. Bitte setzen Sie nicht die Versionen 4.0.0, 4.0.1 und 4.0.2 ein, da diese Probleme in der Behandlung von Cookies hatten, die dazu führten, dass Artikel nicht in den Warenkorb gelegt und Kunden sich nicht an den Shop anmelden konnten.

Wenn dieses Verhalten in Ihrem Shop auftritt und Sie nicht auf die neuere Version von Varnish aktualisieren können, versuchen Sie den folgenden Workaround. Dieser wurde nicht explizit getestet, deshalb prüfen Sie das Verhalten des Shops gründlich, bevor die Änderung in die Produktivumgebung übernommen wird.

Ersetzen Sie in der Konfigurationsdatei :file:`default.vcl` die Zeile 463 |br|
``set beresp.http.Set-Cookie = regsuball(beresp.http.Set-Cookie,\"(, |^)[^@][^,|$]+\",\"\");``
durch diese Zeile |br|
``set beresp.http.Set-Cookie = regsuball(beresp.http.Set-Cookie,\"(, |^)[^@]\",\"\");``

Konfigurationsdateien
^^^^^^^^^^^^^^^^^^^^^
Die beiden Konfigurationsdateien :file:`default.vcl` und :file:`servers_conf.vcl` für die Konfiguration des Reverse Proxys können mit Composer aus einem Repository auf GitHub gezogen werden. Das Paket wird von unserem Satis-Server bereitgestellt. Um es herunterzuladen, müssen per Konsole folgende Composer-Kommandos im Hauptverzeichnis des Shops ausgeführt werden:

.. code::

  composer global config repositories.oxid-esales/varnish-configuration \
    composer https://varnish.packages.oxid-esales.com/

  composer global require oxid-esales/varnish-configuration:^v4.0.0

Auf dieses geschützte Repository kann mit dem Passwort zugegriffen werden, das Shopbetreiber beim Kauf der Hochlastoption erhalten haben. Sollten Probleme auftreten, wenden Sie sich bitte an den Technischen Support.

Im Verzeichnis :file:`/vendor/oxid-esales/varnish-configuration/` befinden sich danach die Dateien :file:`default.vcl` und :file:`servers_conf.vcl.dist`. Benennen Sie die Datei :file:`servers_conf.vcl.dist` in :file:`servers_conf.vcl` um und ersetzen Sie darin folgende Platzhalter:

* ``<my_shop_hostname>`` - IP/Hostname des Backend-Servers vom Shop
* ``<my_shop_IP>`` - IP des Nodes, der den Cache löschen darf

Kopieren Sie die Dateien in das Verzeichnis :file:`/etc/varnish`. Wurden diese Dateien in Ihrem System bereits angepasst, müssen Sie die Inhalte der Dateien manuell zusammenführen. Starten Sie danach Apache und Varnish neu.

:command:`/etc/init.d/apache2 stop` |br|
:command:`/etc/init.d/varnish restart` |br|
:command:`/etc/init.d/apache2 start`

SSL-Verschlüsselung
^^^^^^^^^^^^^^^^^^^
Varnish verarbeitet Anfragen aus dem Web, die das HTTP-Protokoll verwenden. Verschlüsselte Anfragen mit HTTPS-Protokoll können durch den Reverse Proxy nicht umgesetzt werden. Da der OXID eShop auf SSL-Verschlüsselung umschalten kann, sobald Benutzerdaten übertragen werden, beispielsweise bei Registrierung, Anmeldung oder im Warenkorb, muss dafür eine separate Lösung geschaffen werden. Es gibt dafür aktuell zwei Möglichkeiten. Zum einen können Anfragen mit HTTPS-Protokoll direkt an den Server mit dem OXID eShop gesendet werden. Das muss mit Server-Tools umgesetzt werden. Zum anderen kann ein Load Balancer eingesetzt werden, welcher Anfragen über HTTP, Port 80 an Varnish und über HTTPS, Port 443 direkt zum OXID eShop leitet.


.. Intern: oxbacb, Status: