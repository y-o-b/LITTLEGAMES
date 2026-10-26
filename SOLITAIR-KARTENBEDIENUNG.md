# Solitär: Karten auswählen und verschieben

Dieses Dokument beschreibt den Karten-Eingabeweg in `solitaire.html`.

## Bezeichnungen der Stapel

- **Ziehstapel** (`stock`): verdeckter Kartenstapel oben links. Ein Tap zieht die oberste Karte auf die Ablage. Ist er leer, wird die Ablage zurückgedreht.
- **Ablagestapel** (`waste`): Stapel neben dem Ziehstapel mit der zuletzt gezogenen, offenen Karte.
- **Zielstapel** (`foundations`): die vier Stapel oben rechts. Auf ihnen werden die Karten nach Farbe vom Ass aufwärts abgelegt.
- **Spielspalten** bzw. **Tableau** (`tableau`): die sieben Spalten im Hauptspielfeld. Hier werden Kartenfolgen absteigend mit wechselnden Farben gebaut.

## Datenmodell der Auswahl

Eine Auswahl wird in `selected` als `{ source, index, cardIndex }` gespeichert:

- `source`: `tableau`, `waste` oder `foundation`
- `index`: Spalten- bzw. Stapelindex
- `cardIndex`: Position der Karte innerhalb einer Tableau-Spalte

`picked()` löst daraus die tatsächlich bewegte Karte oder Kartenfolge auf. Im Tableau umfasst die Auswahl die angeklickte offene Karte und alle offenen Karten darunter. Ablage und Zielstapel liefern jeweils nur ihre oberste Karte.

## Klick und Touch-Tap

Alle sichtbaren Karten werden von `cardButton()` erzeugt. Der Klick-Handler arbeitet ohne verzögerten Einzelklick:

1. Ein erster Tap wählt die Karte bzw. Tableau-Folge aus und zeichnet die Auswahl neu.
2. Ein weiterer Tap auf dieselbe Karten-ID innerhalb von 500 ms ruft `autoMove()` auf. Die Karte wird auf einen erlaubten Zielstapel bewegt; falls keiner passt, wird sie auf einen passenden Tableau-Stapel gesetzt.
3. Ist bereits eine andere Karte ausgewählt, bewegt ein Tap auf eine Tableau-Karte die Auswahl in deren Spalte.
4. Ein Tap auf einen Grundstapel verwendet dessen Capture-Phase-Handler. So wird auch ein belegter Grundstapel als Ziel erkannt, obwohl darin selbst ein Karten-Button liegt.

Touch-Taps verwenden denselben Click-Handler wie Mausklicks. Ein erfolgreicher Touch-Drag setzt `suppressClick`, damit der vom Browser zusätzlich erzeugte Click nach dem Loslassen den Zug nicht ein zweites Mal ausführt.

## Ziehstapel

Ein Tap auf den Ziehstapel ruft `drawStock()` sofort auf. `lastStockTapAt` fasst einen zweiten Tap innerhalb von 500 ms zusammen, sodass ein Doppeltap nicht zwei Karten zieht. Ist der Vorrat leer, wird die Ablage zurück in den Vorrat gedreht.

## Regeln und Zielprüfung

`moveToTableau()` prüft vor einer Änderung:

- Die ausgewählte Folge ist absteigend und wechselt zwischen Rot und Schwarz (`validSequence()`).
- Auf eine leere Spalte darf nur ein König.
- Auf eine belegte Spalte darf eine Karte, wenn sie eine Rangstufe niedriger und von der Gegenfarbe ist.

`moveToFoundation()` akzeptiert nur eine einzelne Karte. Ein leerer Grundstapel nimmt ein Ass auf; andernfalls müssen Farbe und Folge stimmen. Dieselbe Regel wird für die automatische Ablage durch `canPlaceOnFoundation()` verwendet.

Ungültige Züge ändern den Spielstand nicht. Gültige Züge sichern vorher den Zustand für „Rückgängig“, aktualisieren die Zähler und zeichnen das Spielfeld neu.

## Desktop-Drag & Drop

Karten-Buttons sind `draggable`. Beim `dragstart` wird die Karte bzw. Folge in `selected` übernommen. Tableau-Spalten und Grundstapel erlauben `dragover` und verarbeiten den Drop über `moveToTableau()` bzw. `moveToFoundation()`.

## Touch-Drag & Drop

Auf Touch-Geräten wird ein Pointer-Drag direkt an der Originalkarte ausgeführt:

1. `pointerdown` merkt Ausgangspunkt und bewegte offene Folge.
2. Nach mindestens 8 px Bewegung werden alle Karten der Folge gemeinsam mit dem Finger verschoben und über andere Karten gelegt.
3. `pointerup` ermittelt das Ziel unter dem Finger, übergibt die Ausgangspositionen für die Animation und führt den passenden Zug aus.
4. `pointercancel` setzt temporäre Styles zurück.

## Verschiebe- und Umdreh-Animation

`animateCardMoves()` verwendet die gespeicherten Ausgangsrechtecke und die neuen Kartenpositionen (FLIP). Die Web-Animations-API bewegt die Karten 600 ms lang; währenddessen erhalten sie einen hohen `z-index`. Bei Kartenfolgen wird jede Karte einzeln animiert.

Wird durch einen Tableau-Zug eine verdeckte Karte frei, bleibt sie während der Verschiebung verdeckt. Erst nach Ende der Verschiebe-Animation setzt `revealAfterMove()` `faceUp` und startet die Umdreh-Animation.

## Relevante Funktionen

| Funktion | Aufgabe |
| --- | --- |
| `cardButton()` | Karten-UI, Auswahl, Doppeltap, native Drag- und Touch-Pointer-Events |
| `picked()` | Ermittelt die bewegte Karte bzw. Folge |
| `autoMove()` | Findet für Doppeltap ein erlaubtes Ziel |
| `canPlaceOnFoundation()` | Prüft die Grundstapel-Regel |
| `moveToTableau()` | Validiert und führt Tableau-Züge aus |
| `moveToFoundation()` | Validiert und führt Grundstapel-Züge aus |
| `animateCardMoves()` | Animiert Karten vom Ursprung zum Ziel |
| `revealAfterMove()` | Deckt nach der Bewegung eine freigelegte Karte auf |
