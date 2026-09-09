# Memex

Deutsch · [English](README.md)

Ein persönliches Wissenssystem, das ein LLM aufbaut und pflegt. Markdown als Speicher, eine HTML-Datei als
Ansicht, ein Python-Skript für PDFs. Kein Framework, keine Datenbank, keine Vektorsuche.

Der Name Memex verweist auf Vannevar Bushs Idee von 1945: ein persönlicher, gepflegter Wissensspeicher, in dem
die Verbindungen zwischen Dokumenten so wertvoll sind wie die Dokumente selbst. Bush konnte nicht lösen, wer die
Pflege übernimmt. Das LLM kann es.

Memex ist das Ganze: Eingang, Rohquellen, Schema, Skripte, Ansicht. Das Wiki ist sein Kern, die Sammlung der
Seiten, die das LLM aus den Quellen schreibt und pflegt. Wenn im Folgenden vom Wiki die Rede ist, ist diese
Schicht gemeint.

Diese Datei ist eine Ideendatei. Sie ist dafür gedacht, in einen LLM-Agenten kopiert zu werden, egal welchen:
Claude Code, Codex, Gemini CLI, OpenCode, Aider oder was du sonst nutzt. Sie beschreibt, wie ich mein System
gebaut habe, warum, und was die drei Skripte tun. Dein Agent kann daraus in ein bis zwei Sitzungen eine eigene
Fassung bauen. Die Datei steht für sich und braucht nichts anderes.

## Die Idee

Die übliche Art, mit einem LLM über Dokumente zu arbeiten, ist das Nachschlagen: Man lädt Dateien hoch, das
Modell sucht bei jeder Frage passende Ausschnitte und antwortet daraus. Das funktioniert, aber nichts sammelt sich
an. Jede Frage beginnt bei null. Eine Frage, die fünf Dokumente verbindet, muss das Modell jedes Mal neu
zusammensetzen.

Der andere Weg: Das LLM baut aus den Quellen ein dauerhaftes Wiki auf. Jede neue Quelle wird einmal gelesen,
verdichtet und in bestehende Seiten eingearbeitet. Querverweise, Widersprüche und Zusammenfassungen sind schon
da, wenn die Frage kommt. Antworten, die etwas wert sind, werden als Seiten ins Wiki abgelegt. Das Wiki wird mit
jeder Quelle und jeder Frage reicher.

Ich schreibe das Wiki nicht selbst. Ich liefere Quellen, stelle Fragen und lese. Das LLM macht die Buchhaltung:
zusammenfassen, verlinken, einordnen, Inhaltsverzeichnis und Verlauf pflegen. Menschen geben Wikis auf, weil die
Pflege schneller wächst als der Nutzen. Ein LLM wird nicht müde, vergisst keinen Querverweis und kann fünfzehn
Seiten in einem Durchgang anfassen.

Das Muster ist nicht fachgebunden. Bei mir sind die Quellen Fachbücher mit über tausend Seiten, Behördenberichte
und Forschungsberichte. Es funktioniert für jedes Gebiet, in dem sich Wissen über Monate aus vielen Dokumenten
aufbaut: Forschung, ein Buch beim Lesen begleiten, Normen und Regelwerke, Projektwissen, Hobbythemen.

## Vier Entscheidungen

Ich wollte etwas, das im Browser läuft, keine Installation braucht und mit so wenig Werkzeug wie möglich
auskommt. Daraus ergaben sich vier Entscheidungen:

1. **Markdown bleibt die Wahrheit.** Das LLM schreibt Markdown, nichts anderes. Die HTML-Datei wird daraus
   erzeugt und nie von Hand angefasst.
2. **Eine HTML-Datei ist die Ansicht.** Ein kurzes Python-Skript setzt alle Wiki-Seiten zu einer einzigen
   `index.html` zusammen, mit Navigation nach Seitentyp, Rückverweisen, Volltextsuche und Formeln. Sie läuft
   per Doppelklick aus einem Ordner und genauso auf einem Webdienst. Kein Website-Generator, weil die einen
   Server brauchen und hunderte Pakete mitbringen.
3. **PDFs werden nur zu Markdown.** Docling wandelt um, ein Skript teilt an Überschriften. Kein JSON, keine
   Bilder, kein Zerlegen nach Textlänge. Das Original-PDF bleibt neben dem Markdown liegen, damit das LLM bei
   Formeln und Abbildungen die Seite selbst anschauen kann.
4. **Ein Python-Umfeld, drei Skripte.** Umwandeln, erzeugen, prüfen. Sonst nichts.

## Aufbau

Drei Schichten mit klaren Zuständigkeiten, dazu die Ansicht:

```
memex/
├── <schema>.md        Schema: Regeln, Seitentypen, Abläufe. Der Agent liest es bei jedem Start.
├── inbox/             Eingang: unverarbeitete PDFs. Leer heisst: alles verarbeitet.
├── raw/<quelle>/      Rohquellen: PDF, Volltext, Inhaltsverzeichnis, Abschnitte. Nur lesen.
├── wiki/              Das Wiki: index.md, log.md, home.md und alle Seiten. Flach, kein Unterordner.
├── tools/             convert.py (PDF → raw), build.py (wiki → site), lint.py (Prüfbericht)
└── site/index.html    Die ganze Ansicht in einer Datei.
```

**Rohquellen.** Du legst ab, das LLM liest, niemand ändert. Jede Quelle bekommt ein Kürzel (`handbuch-xy`,
`norm-123`) und einen Ordner. Darin liegt das PDF, ein `_volltext.md` mit Seitenmarken als Archiv, ein
`_index.md` mit der Kapitelliste und pro Abschnitt eine Datei mit Titel, Quelle und PDF-Seitenbereich im
Kopfblock. Bei Büchern mit hunderten Abschnitten kommt eine `_abschnitte.md` dazu, die das LLM gezielt
durchsucht, damit das Inhaltsverzeichnis kurz bleibt.

**Wiki.** Das LLM schreibt, du liest. Flach, eine Datei pro Seite, der Dateiname ist der Linkname. Jede Seite
beginnt mit einem Kopfblock (Frontmatter): Typ (`quelle`, `konzept`, `entitaet`, `projekt`, `antwort`, `meta`),
Reifegrad (`seedling`, `growing`, `evergreen`), Datum und die Kürzel der Quellen, auf die sie sich stützt. Links
als `[[dateiname]]`. Zitate mit Werk, Abschnitt und PDF-Seite.

```yaml
---
title: Titel der Seite
type: konzept
status: growing
updated: 2026-09-09
sources: [handbuch-xy, norm-123]
---
```

**Schema.** Die wichtigste Datei. Jeder Agent hat einen eigenen Dateinamen dafür, den er beim Start automatisch
liest. Sie sagt dem LLM, was wo liegt, wie Seiten aussehen und welche Schritte jeder Ablauf hat. Sie wächst mit
der Erfahrung; jede Änderung daran bekommt einen Eintrag im Verlauf.

**Zwei Sonderdateien.** `wiki/index.md` ist das Inhaltsverzeichnis: jede Seite mit einer Zeile Beschreibung,
nach Typ gruppiert. Das LLM liest es vor jeder Antwort und pflegt es bei jeder Änderung. `wiki/log.md` ist der
Verlauf: Es wird nur angehängt, nie geändert, ein Eintrag pro Vorgang mit Datum und Art
(`## [2026-09-09] einarbeiten | Titel`), darunter zwei bis fünf Zeilen. Das Inhaltsverzeichnis ersetzt bei
einigen hundert Seiten jede Suchtechnik, der Verlauf gibt dem LLM den Zusammenhang der letzten Sitzungen.

## Die drei Skripte

Alles Werkzeug sind drei Python-Dateien, zusammen wenige hundert Zeilen, geschrieben vom Agenten in einer
Sitzung. Einzige Abhängigkeiten: `docling` (IBM, MIT-Lizenz) für PDFs und `markdown` für die Ansicht. Dieser
Abschnitt ist für den Agenten geschrieben und entsprechend technisch.

**`convert.py`: PDF → `raw/<kürzel>/`.** Aufruf mit PDF-Pfad, Kürzel und Titel.

1. Docling mit `do_ocr=False` (wenn die PDFs eine Textebene haben), Tabellen im Modus «accurate»,
   Formel-Anreicherung aus, Gerät `auto`, keine Bilder.
2. Ausgabe des ganzen Dokuments als Markdown mit `page_break_placeholder`. Die Marke steht nur zwischen
   nichtleeren Seiten, deshalb wird die Seitenfolge aus `doc.iterate_items()` abgeleitet (Seitennummer jedes
   Elements in Reihenfolge). Ergebnis: `_volltext.md` mit `<!-- SEITE n -->` vor jeder Seite. Damit lässt sich die
   Teilung jederzeit in Sekunden wiederholen (`--split-only`).
3. Teilen: Jede Überschrift wird ein Atom mit Text bis zur nächsten. Tiefe aus der Nummerierung («2» → 2,
   «2.4» → 3, «2.4.5» → 4, «1.2.7-6» → 5), unnummerierte Überschriften gelten als Geschwister der zuletzt
   gesehenen nummerierten, Aufzählungen «2. Beispiel» zählen nicht. Atome werden an Überschriften bis `--level`
   (Standard 2) gruppiert. Gruppen über `--max-chars` (40'000) werden rekursiv an tieferen Überschriften
   geteilt, notfalls an Leerzeilen nach Grösse. Gruppen unter `--min-chars` (2'500) wandern an ihren Nachbarn:
   Inhaltsverzeichnisse und kurze Kapitelköpfe nach vorne in den Folgeabschnitt, alles andere an den Vorgänger.
   Sachverzeichnis, Wörterbücher und Inserenten fallen weg.
4. Schreiben: `NNN_titel.md` mit Kopfblock `title`, `source`, `pages` (PDF-Seitenbereich), H1 und Text.
   Steuerzeichen aus kaputten Formeln werden ersetzt und die Datei mit `formulas_damaged: true` markiert.
   `_index.md` mit Kopf (Werk, PDF-Pfad, Seitenzahl, Datum, Parameter), Kapitelliste (oberste nummerierte
   Ebene mit mindestens zehn Einträgen) und der Abschnittstabelle; bei über 200 Abschnitten wandert die Tabelle
   nach `_abschnitte.md`.
5. Das PDF wird von `inbox/` nach `raw/<kürzel>/` verschoben.

**`build.py`: `wiki/*.md` → `site/index.html`.** Liest alle Seiten, liest den Kopfblock mit einem kleinen
eigenen Parser aus (Schlüssel, Listen in eckigen Klammern), schützt Formeln (`$…$`, `$$…$$`) vor dem
Markdown-Parser durch Platzhalter aus reinem ASCII, ersetzt `[[ziel|text]]` durch Links auf `#ziel` (Auflösung
über Dateiname oder Titel, fehlende Ziele als gestrichelter Hinweis), wandelt Markdown mit den Erweiterungen
Tabellen, Fussnoten und Codeblöcke um, berechnet Rückverweise. Ausgabe: eine HTML-Datei mit eingebettetem CSS und
JavaScript. Seitenleiste nach Typ, jede Seite als verstecktes `<section id="slug">`, Seitenwechsel über
`hashchange`, Suche als Teilstring-Filter über den Text aller Seiten mit Trefferauszug, Metazeile
(Typ · Reifegrad · Datum · Quellen), Rückverweise als Fusszeile, KaTeX vom CDN nur wenn eine Seite ein `$`
enthält, helles und dunkles Design über `prefers-color-scheme`. Meldet fehlende Linkziele.

**`lint.py`.** Prüft jede Wiki-Seite: Kopfblock vorhanden, Typ und Reifegrad aus der erlaubten Menge, Datum im
Format `YYYY-MM-DD`, Dateiname klein-ascii-bindestrich, Quellenkürzel mit existierendem Ordner, tote Links,
Waisen (Seiten, auf die nur Inhaltsverzeichnis oder Verlauf verweisen), Seiten, die im Inhaltsverzeichnis
fehlen, Verlaufseinträge ohne Standardformat. Fehler geben Exit 1, Hinweise nur Text.

## Abläufe

**Eingang.** Ich lege ein PDF in `inbox/` und sage «neue Datei in inbox». Das LLM schlägt Kürzel und Titel vor,
wandelt um, prüft das Inhaltsverzeichnis der Quelle und verschiebt das PDF in seinen Quellordner.

**Einarbeiten.** «einarbeiten handbuch-xy». Das LLM liest Inhaltsverzeichnis und Abschnitte der Quelle,
bespricht die Kernaussagen mit mir, schreibt eine Quellenseite, legt Konzept- und Entitätsseiten an oder
aktualisiert sie, markiert Widersprüche zu bestehenden Seiten, ergänzt Inhaltsverzeichnis und Verlauf, erzeugt
die Ansicht. Eine Quelle darf zehn bis fünfzehn Seiten berühren. Bücher werden nicht am Stück eingearbeitet. Sie
bekommen eine Quellenseite mit Kapitelübersicht, Kapitel werden gelesen, wenn eine Frage sie braucht. So wächst
das Wiki dort, wo man arbeitet.

**Fragen.** Ich frage. Das LLM liest zuerst `wiki/index.md`, dann die passenden Seiten, bei Bedarf die
Rohquellen. Antwort mit Zitaten. Ist die Antwort etwas wert (ein Vergleich, eine Analyse, eine
Entscheidungsgrundlage), wird sie als `antwort`-Seite im Wiki abgelegt und von den betroffenen Konzeptseiten
verlinkt. So verschwinden Erkundungen nicht im Chatverlauf.

**Prüfen.** `tools/lint.py` findet tote Links, Waisen, Seiten ohne Quellen und Seiten, die im
Inhaltsverzeichnis fehlen. Das LLM prüft dazu Widersprüche zwischen Seiten, veraltete Aussagen, Konzepte ohne
eigene Seite und fehlende Querverweise. Befunde landen als Aufgabenliste auf der Startseite.

**Ende einer Sitzung.** Ansicht erzeugen, Verlauf prüfen. Wer das Wiki versioniert, hält den Stand jetzt fest,
mit einem Vermerk nach Art des Vorgangs (`einarbeiten:`, `frage:`, `pruefen:`, `schema:`).

## PDF zu Markdown: was ich gelernt habe

Das war der aufwendigste Teil, und die Fehler sind lehrreich.

- **Nicht nach Textlänge zerlegen.** Mein erster Versuch mit einem Zerleger nach Wortzahl machte aus einem Buch
  zehntausende Dateien, oft ein Absatz pro Datei. Das Inhaltsverzeichnis dazu war so lang wie das Buch und als
  Einstieg unbrauchbar. Jetzt gebe ich das ganze Dokument einmal als Markdown aus und teile an Überschriften. Ziel
  sind Abschnitte, die ein LLM in einem Zug liest, pro Buch einige hundert Dateien und ein Inhaltsverzeichnis,
  das auf eine Bildschirmseite passt.
- **Docling setzt alle Überschriften auf eine Ebene.** Die Hierarchie steckt in der Nummerierung. Das Skript
  leitet die Tiefe daraus ab, teilt zu grosse Abschnitte feiner, hängt kleine an ihren Nachbarn und lässt
  Sachverzeichnis und Wörterbuch weg. Buchteile ohne Nummer erkennt Docling oft nicht als Überschrift; die
  Kapitelliste gleicht das aus.
- **Formel-Anreicherung aus.** Docling kann Formeln als Bild lesen und zu LaTeX machen. Auf einem Rechner ohne
  Grafikkarte dauert das bei einem dicken Buch die ganze Nacht und ist am Morgen nicht fertig. Ohne Anreicherung
  ist dasselbe Buch in einer Kaffeepause umgewandelt. Formeln aus der Textebene des PDF sind ohnehin oft
  beschädigt. Regel im Schema: Bei Formeln liest das LLM die PDF-Seite direkt. Deshalb bleibt das PDF im
  Quellordner.
- **Ausgabe einmal, nicht pro Seite.** Die seitenweise Ausgabe wird mit jeder Seite langsamer und kostet bei
  einem Buch mehr Zeit als die Umwandlung selbst. Die Ausgabe mit Seitenmarke in einem Durchgang dauert Sekunden.
  Die Seitenzuordnung kommt aus der Reihenfolge der Dokumentelemente und ist exakt.
- **Alle Seitenzahlen sind PDF-Seiten**, nicht die gedruckte Paginierung. Das steht in jedem Inhaltsverzeichnis,
  damit Zitate nachschlagbar bleiben.

Ohne Texterkennung (OCR) und ohne Formel-Anreicherung ist die Umwandlung schnell: ein Bericht in Sekunden, ein
Handbuch mit tausenden Seiten in Minuten.

## Die Ansicht

Eine HTML-Datei von einigen hundert Kilobyte, die mit dem Wiki wächst. Seitenleiste nach Typ, Suche über alle
Seitentexte (Taste `/`), Metazeile mit Typ, Reifegrad, Datum und Quellen, Rückverweise unter jeder Seite,
Formeln über KaTeX, helles und dunkles Design nach Systemeinstellung. Seitenwechsel über `#seitenname`, also
ohne Server.

Grenzen, die ich in Kauf nehme: keine Netzansicht der Verknüpfungen (Rückverweise reichen mir), keine Bearbeitung
im Browser (das LLM schreibt), Formeln brauchen einmalig Internet für KaTeX.

Die Datei läuft per Doppelklick aus jedem Ordner, auch aus einem synchronisierten Cloud-Laufwerk. Weil sie eine
einzelne statische Datei ist, lässt sie sich bei jedem Webdienst ablegen und mit einer Anmeldung schützen, wenn
man sie unterwegs will. Die Rohquellen gehören nicht dorthin, weil sie geschützte Werke enthalten können. Das
Wiki selbst enthält nur Kurzzitate.

## Was ich bewusst weggelassen habe

- Kein JSON, kein zweites Dokumentformat, keine Bildausgabe.
- Kein Website-Generator, kein Node.
- Keine Vektorsuche. Inhaltsverzeichnis plus Suche in der Ansicht reichen bis mehrere hundert Seiten, danach
  ein kleines Suchskript.
- Kein Ordnerbaum im Wiki. Flach, mit Typ im Kopfblock. Weniger Pfade, weniger tote Links.
- Keine Tagesnotizen, keine Projektverwaltung, bis ich sie brauche. Das Muster ist dafür offen.

## So baust du dein eigenes

1. Leg einen Ordner an, kopiere diese Datei hinein und gib sie deinem Agenten. Sag ihm, welches Fachgebiet und
   welche Quellen du hast und in welcher Sprache das Wiki sein soll.
2. Lass den Agenten zuerst das Schema schreiben, in der Datei, die er beim Start liest: Schichten und Ordner,
   Seitentypen und Kopfblock, die vier Abläufe mit Prüfliste, Zitierregeln, Ende einer Sitzung. Das Schema ist
   die wichtigste Datei und wächst mit jeder Sitzung.
3. Lass ihn die drei Skripte nach der Beschreibung oben bauen und an einem kleinen PDF testen. Python 3.10 oder
   neuer in einer virtuellen Umgebung ausserhalb von Cloud-Laufwerken, darin `docling` und `markdown`.
4. Erstes PDF in `inbox/`, «neue Datei in inbox». Inhaltsverzeichnis der Quelle anschauen: Sind die Abschnitte
   brauchbar gross, ist die Kapitelliste plausibel? Wenn nicht, die Teilung mit anderer Überschriftenebene oder
   anderer Mindestgrösse wiederholen. Das geht aus dem gespeicherten Volltext in Sekunden, ohne das PDF neu
   umzuwandeln.
5. Ansicht erzeugen, `site/index.html` im Browser öffnen. Die HTML-Datei zeigt immer den Stand der letzten
   Erzeugung. Deshalb lässt der Agent das Skript am Ende jeder Sitzung laufen, sonst fehlen die neuen Seiten in
   der Ansicht. Das gehört als fester Schritt ins Schema.
6. Erste Quelle einarbeiten, erste Frage stellen, erste Antwort als Seite ablegen. Nach ein paar Sitzungen das
   Schema überarbeiten. Es wird deins.

## Hinweis

Diese Datei beschreibt ein Muster, nicht einen Standard. Die Kürzel, die Seitentypen, die Abschnittsgrössen,
Versionierung und Veröffentlichung: alles ist an meinen Fall angepasst und austauschbar. Der Agent soll die Datei
lesen und mit dir zusammen die Fassung bauen, die zu deinen Quellen, deinem Werkzeug und deiner Arbeitsweise
passt.
