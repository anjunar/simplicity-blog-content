In dieser Reihe entwickeln wir aus einem leeren Repository einen Blog, der schließlich unter einer eigenen Domain läuft. Auf dem Server und im Browser verwenden wir Scala, für die Datenhaltung PostgreSQL. Serverseitiges Rendering liefert lesbare Seiten, bevor der Browser die Interaktivität übernimmt.

Das begleitende Projekt heißt `anjunar-blog-tutorial`. Jeder Implementierungsartikel erweitert dasselbe Projekt und erklärt, wie sich das Ergebnis starten und überprüfen lässt.

Dieser erste Artikel legt fest, was wir bauen und wie die Teile zusammenwirken. Im nächsten Artikel legen wir das Projekt an.

## Was die fertige Anwendung können soll

Besucher können Artikel lesen, durchsuchen und nach Tags filtern. Benutzer können sich registrieren und anmelden. Administratoren erhalten einen Editor, mit dem sie Artikel schreiben, ändern und veröffentlichen.

Ein Artikel besitzt einen Titel, eine stabile URL, einen Veröffentlichungsstatus und Inhalte mit formatiertem Text, Codeblöcken und Bildern. Entwürfe bleiben privat. Erst die Veröffentlichung macht einen Artikel auf der öffentlichen Seite zugänglich.

Oberfläche und Artikel unterstützen Englisch und Deutsch; Englisch ist zunächst die Hauptsprache. Übersetzungen der Oberfläche behandeln wir getrennt von übersetzten Artikeln: Ein übersetzter Speichern-Button und die deutsche Fassung eines Artikels sind unterschiedliche Daten.

Die Anwendung muss über eine Entwicklungssitzung hinaus funktionieren. Dazu brauchen wir Datenbankmigrationen, ein bereitstellbares Paket, HTTPS, Backups und ausreichend Protokollierung, um fehlgeschlagene Anfragen untersuchen zu können.

Die technische Referenz ist Anjunar Stack. Im Tutorial bauen wir eine einzelne Website. Deshalb benötigt ihr Datenmodell weder Tenant-IDs noch einen Tenant-Kontext.

## Der Stack und die Aufgaben seiner Bestandteile

Scala kommt in zwei Laufzeitumgebungen zum Einsatz: Das Backend läuft auf der JVM, während Scala.js das Frontend nach JavaScript übersetzt.

Die wichtigsten Bestandteile übernehmen folgende Aufgaben:

| Komponente | Aufgabe |
| --- | --- |
| sbt | Module und Abhängigkeiten definieren, Code kompilieren, Tests ausführen und die Anwendung paketieren. |
| Undertow | HTTP-Anfragen annehmen und die Anwendung bereitstellen. |
| RESTEasy | REST-Anfragen an annotierte Ressourcenmethoden weiterleiten und HTTP-Antworten erzeugen. |
| Weld | Anwendungskomponenten mittels CDI Dependency Injection erzeugen und verbinden. |
| Hibernate und PostgreSQL | Persistente Entitäten abbilden und Anwendungsdaten speichern. |
| Agroal und Narayana | Datenbankverbindungen und Transaktionsgrenzen verwalten. |
| Scala.js UI | Die Oberfläche aufbauen, reaktiven Zustand binden, Formulare verarbeiten und Routen mit Komponenten verbinden. |
| GraalJS | Das Bundle für serverseitiges Rendering innerhalb des JVM-Prozesses ausführen. |

Außerdem verwenden wir den JSON-Mapper und den Hibernate DDL Manager von Anjunar. Der Mapper verbindet Entity-Schema, Zugriffsregeln und JSON-Darstellung. Der DDL Manager entwickelt das Datenbankschema anhand der Entity-Mappings weiter.

Diese Bibliotheken sind Abhängigkeiten der Anwendung. Wir beziehen ihre veröffentlichten Versionen aus Maven Central. Jedes Kapitel ergänzt die Konfiguration für den jeweils neuen Bestandteil.

Das Frontend erzeugt zwei JavaScript-Bundles: eines für das Rendering auf dem Server und eines für den Browser. Beide nutzen dieselben UI-Komponenten, aber getrennte Einstiegspunkte für ihre Laufzeitumgebungen.

## Der Weg einer Leser-Anfrage

Stellen wir uns einen veröffentlichten Artikel mit dem Titel *Mein erster Artikel* vor. Ein Besucher öffnet seine öffentliche URL.

1. **Undertow empfängt die Seitenanfrage.** Die Anwendung wählt die Route und beginnt, sie mit GraalJS zu rendern.
2. **Die Seite lädt ihre Daten über die REST-API.** Der Endpunkt sucht den Artikel in PostgreSQL und prüft, ob der Besucher ihn lesen darf.
3. **Die API liefert die öffentliche Darstellung.** Die Antwort enthält die für diese Ansicht benötigten Felder. Ein Entity-Graph bestimmt die ausgewählten Felder; Zugriffsregeln legen fest, welche davon ausgegeben werden dürfen.
4. **Das Frontend rendert die Seite auf dem Server.** Das Ergebnis enthält den HTML-Inhalt des Artikels, Seitenmetadaten und den anfänglichen Anwendungszustand für den Browser.
5. **Der Browser übernimmt.** Er lädt sein JavaScript-Bundle und hydriert das vorhandene HTML. Dabei verbindet er die Oberfläche mit ihrem Zustand und den Ereignisbehandlern.

Daraus ergibt sich eine überprüfbare Anforderung: Das anfängliche HTML muss den Artikel enthalten. Nach dem Start des Browsers müssen sich Server und Browser über Route, Sprache und Daten einig sein.

Ein unveröffentlichter Artikel nimmt einen anderen Weg. Die API muss seine Sichtbarkeit prüfen, bevor Inhalte die öffentliche Seite erreichen. Diese Entscheidung gehört auf den Server, auch wenn die Oberfläche nicht verfügbare Aktionen ebenfalls ausblendet.

## Der Weg beim Speichern im Editor

Beim Schreiben eines Artikels kommen weitere Aufgaben hinzu.

Der Editor lädt ein Frontend-Modell, das die von der API bereitgestellten Felder spiegelt. Formularelemente binden sich an dieses Modell. Wenn der Autor den Titel ändert und auf Speichern klickt, sendet der Client die bearbeiteten Daten und die ursprünglich geladene Version.

Der Server muss feststellen, wer die Anfrage stellt, ob diese Person den Artikel bearbeiten darf und welche Felder sie ändern darf. Ein `PreparedChange` hält die vorgeschlagenen Änderungen zurück, bis der Controller Berechtigungen und fachliche Bedingungen geprüft hat.

Nach dem Anwenden der erlaubten Änderungen validiert der Server den resultierenden Zustand und schließt die Transaktion ab. Bei Erfolg erhält der Editor die gespeicherten Werte und die aktuelle Version zurück.

Fehler brauchen ebenso eindeutiges Verhalten:

- Ein ungültiger Titel erzeugt einen Feldfehler, den das Formular anzeigen kann.
- Der Server weist eine unberechtigte Änderung zurück.
- Ein Speicherversuch mit einer veralteten Version führt zu einem Konflikt, statt eine fremde Änderung stillschweigend zu überschreiben.

Diese Fälle gehören zur Implementierung und zu ihren Tests. Von ihnen hängt ab, ob Autoren der Anwendung ihre Arbeit anvertrauen können.

## Die Bestandteile aufeinander abstimmen

Ein Artikel taucht an mehreren Stellen auf: im Datenbank-Mapping, im Entity-Schema, in der REST-Antwort, im Frontend-Modell und im Formular. Ein neues Feld muss durch diese ganze Kette geführt werden.

Wenn wir beispielsweise einen Veröffentlichungsstatus einführen, definieren wir seinen gespeicherten Wert, entscheiden, wer ihn ändern darf, nehmen ihn in die passende API-Antwort auf und binden ihn im Editor. Außerdem prüfen wir, ob eine Änderung die öffentliche Sichtbarkeit korrekt beeinflusst.

Wir organisieren den Code nach diesen Aufgaben. Feature-Module enthalten Domänenlogik, REST-Endpunkte sowie Frontend-Modelle und Aktionen. Gemeinsame Infrastruktur gehört in Plattformmodule. Die Anwendung setzt daraus die laufende Website zusammen.

Am Anfang genügt ein einziges Backend-Modul. Weitere Modulgrenzen fügen wir hinzu, sobald ihre Aufgaben konkret werden.

## Der Weg durch die Artikelreihe

Unser erstes Etappenziel ist klein: den Server lokal starten, einen Artikel in PostgreSQL speichern und ihn über REST abrufen.

Danach bauen wir die Leseroberfläche, verbinden sie mit der API und ergänzen Konten, Berechtigungen und Bearbeitungsfunktionen. Suche, Bilder, strukturierte Inhalte und Übersetzungen erweitern die bereits funktionierende Anwendung.

Anschließend ergänzen wir serverseitiges Rendering und Hydration, vervollständigen die öffentlichen Seiten mit Metadaten und Feeds und paketieren die Anwendung für das Deployment.

Jeder Implementierungsartikel nennt die geänderten Dateien, zeigt den wichtigen Code und liefert einen Befehl oder eine Anfrage zum Prüfen des Ergebnisses. Veröffentlichte Artikel werden an bestimmte Commits oder Tags gebunden, damit ihre Beispiele auch bei wachsendem Projekt reproduzierbar bleiben.

Im nächsten Artikel geht es um die Entwicklungswerkzeuge, `build.sbt` und das erste Backend-Modul. Das Ziel ist einfach: die Anwendung starten und eine HTTP-Antwort erhalten.
