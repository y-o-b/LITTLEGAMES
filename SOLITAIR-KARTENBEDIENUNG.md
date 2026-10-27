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

Alle sichtbaren Karten werden von `cardButton()` erzeugt. Maus (Klick) und Touch (`pointerup`) laufen über dieselbe Funktion `tapCard()`, ohne verzögerten Einzelklick:

1. Ein erster Tap wählt die Karte bzw. Tableau-Folge aus und zeichnet die Auswahl neu.
2. Ein weiterer Tap auf dieselbe Karten-ID innerhalb von 500 ms ruft `autoMove()` auf. Die Karte wird auf einen erlaubten Zielstapel bewegt; falls keiner passt, wird sie auf einen passenden Tableau-Stapel gesetzt.
3. Ist bereits eine andere Karte ausgewählt, bewegt ein Tap auf eine Tableau-Karte die Auswahl in deren Spalte.
4. Ein Tap auf einen Grundstapel verwendet dessen Capture-Phase-Handler. So wird auch ein belegter Grundstapel als Ziel erkannt, obwohl darin selbst ein Karten-Button liegt.

Ein Touch-Tap ruft `tapCard()` direkt im `pointerup` auf und setzt `suppressClick`; der danach vom Browser erzeugte Click wird so nicht doppelt verarbeitet. Dasselbe gilt nach einem Touch-Drag.

Die Buttons (Ziehstapel, Rückgängig, Gleiches Deck, Neues Spiel, Fertigspielen) verwenden `onTap()`: Touch löst direkt im `pointerup` aus (der Finger muss noch auf dem Button sein). Grund: Safari verschluckt sonst den ersten Click nach einem Touch-Drag. Nach jedem verarbeiteten Touch-Tap (Karte oder Button) verwirft `noteTouchTap()` Clicks 800 ms lang global. Das verhindert, dass der von Safari nachgeschobene Click ein anderes Element trifft, wenn sich das Layout inzwischen verschoben hat (z. B. „Fertigspielen" verschwindet und „Neues Spiel" liegt unter dem Finger).

Tastatur: Leere Spalten und leere Zielstapel sind per Tab erreichbar und reagieren auf Enter und Leertaste (ausgewählte Karte dorthin legen).

## Ziehstapel

Ein Tap auf den Ziehstapel ruft `drawStock()` sofort auf. `lastStockTapAt` fasst einen zweiten Tap innerhalb von 250 ms zusammen, sodass eine versehentlich doppelt ausgelöste Eingabe nicht zwei Karten zieht; zügiges Weiterblättern bleibt möglich. Ist der Vorrat leer, wird die Ablage zurück in den Vorrat gedreht.

## Regeln und Zielprüfung

`moveToTableau()` prüft vor einer Änderung:

- Die ausgewählte Folge ist absteigend und wechselt zwischen Rot und Schwarz (`validSequence()`).
- Auf eine leere Spalte darf nur ein König.
- Auf eine belegte Spalte darf eine Karte, wenn sie eine Rangstufe niedriger und von der Gegenfarbe ist.

`moveToFoundation()` akzeptiert nur eine einzelne Karte. Ein leerer Grundstapel nimmt ein Ass auf; andernfalls müssen Farbe und Folge stimmen. Dieselbe Regel wird für die automatische Ablage durch `canPlaceOnFoundation()` verwendet.

Ungültige Züge ändern den Spielstand nicht. Gültige Züge sichern vorher den Zustand für „Rückgängig“, aktualisieren die Zähler und zeichnen das Spielfeld neu.

## Drag & Drop (Maus, Touch, Stift)

Es gibt kein natives HTML5-Drag&Drop mehr (`draggable`, `dragstart`, `drop`). Maus, Touch und Stift verwenden dieselbe Pointer-Logik direkt an der Originalkarte, damit das Ziehen auf allen Geräten gleich aussieht:

1. `pointerdown` (bei der Maus nur die linke Taste) merkt Ausgangspunkt und bewegte offene Folge.
2. Nach mindestens 8 px Bewegung werden alle Karten der Folge gemeinsam mit dem Zeiger verschoben (Klasse `dragging`, leicht transparent, über allen anderen Karten). Bei der Maus zeigt der Cursor dabei die greifende Hand (`body.dragging-cards`), und die Textauswahl ist auf dem Spielfeld abgeschaltet.
3. `pointerup` ermittelt das Ziel unter dem Zeiger, übergibt die Ausgangspositionen für die Animation und führt den passenden Zug aus. Ohne gültiges Ziel fliegt die Karte zurück.
4. `pointercancel` setzt temporäre Styles zurück.

Ein Klick ohne Bewegung läuft bei der Maus weiter über `click` (auswählen, Doppelklick = Automatikzug). Nach einem Maus-Drag werden Clicks 250 ms lang verworfen, nach einem Touch-Drag länger (600 bis 1200 ms), weil Safari den Click verzögert nachschiebt.

## Verschiebe- und Umdreh-Animation

`animateCardMoves()` verwendet die gespeicherten Ausgangsrechtecke und die neuen Kartenpositionen (FLIP). Die Web-Animations-API bewegt die Karten 600 ms lang; währenddessen erhalten sie einen hohen `z-index`. Bei Kartenfolgen wird jede Karte einzeln animiert.

Wird durch einen Tableau-Zug eine verdeckte Karte frei, bleibt sie während der Verschiebung verdeckt. Erst nach Ende der Verschiebe-Animation setzt `revealAfterMove()` `faceUp` und startet die Umdreh-Animation.

## Neues Spiel und Austeilen

- Es gibt keinen eingebauten Ersatz für die Deck-Sammlung mehr. Die Spiele kommen ausschließlich aus `solitaire-deals.json`. Lässt sie sich nicht laden, zeigt „Neues Spiel" eine Meldung und ändert nichts. Der nächste Versuch lädt die Sammlung erneut.
- „Neues Spiel" und „Gleiches Deck" (`startNewGame()`) lassen alle 52 Karten von ihrer aktuellen Position (auch aus dem Haufen oder den Zielstapeln) an ihren Platz im neuen Spiel fliegen. Die Reihenfolge entspricht dem Austeilen: Spalte für Spalte reihum, zuletzt der Ziehstapel. Karten, die ihre Seite wechseln, drehen sich unterwegs (Breite skaliert auf nahe 0 und wieder zurück). Während des Austeilens ist das Spielfeld gesperrt. Bei „Bewegung reduzieren" entfällt die Animation.

## Fertigspielen und Spielende

- „Fertigspielen" erscheint, sobald keine verdeckte Karte mehr liegt. `autoFinishGame()` sperrt das Spielfeld bis auf „Neues Spiel" (`lockBoard()`, `inert`); „Neues Spiel" bricht den Ablauf ab. Die Flugkarten verwenden Seitenkoordinaten und bleiben beim Scrollen auf Kurs. Bei „Bewegung reduzieren" (`prefers-reduced-motion`) entfällt die Animation. Die übrigen Karten fliegen einzeln auf ihren Zielstapel: aufsteigend nach Rang, reihum über die vier Stapel (110 ms Takt, 450 ms Flug). Jede Farbe landet auf dem Stapel, der mit ihrem Ass begonnen wurde.
- Nach jedem Sieg, auch durch den letzten regulären Zug, fliegen alle Karten von den Zielstapeln auf einen wilden Haufen (`startHeap()`, Overlay `#heap`). Danach ist nur noch „Neues Spiel" aktiv: Rückgängig und Gleiches Deck sind gesperrt, das Spielfeld zeigt leere Stapel (`renderHeapView()`).
- Wird ein gewonnenes Spiel aus dem Speicher geladen, erscheint der Haufen sofort ohne Animation. „Neues Spiel" räumt den Haufen weg (`clearHeap()`).
- Die Zielpositionen des Haufens werden als Verhältniswerte gespeichert. Bei einer Größenänderung (z. B. Drehen) berechnet `layoutHeap()` Overlay und Karten neu.

## Speichern

- Der Spielstand liegt im `localStorage` (`STORAGE_KEY`). Geschrieben wird bei jeder Zustandsänderung (`updateStatus()`) sowie beim Verbergen oder Verlassen der Seite (`visibilitychange`, `pagehide`), nicht mehr jede Sekunde und nicht bei jedem Neuzeichnen.
- Der Rückgängig-Verlauf ist auf `HISTORY_LIMIT` (200) Einträge begrenzt, damit der gespeicherte Stand nicht unbegrenzt wächst.
- Beim Laden prüft `validState()`, dass der Stand (und jeder Verlaufseintrag) genau die 52 Karten enthält und jede Karte zu ihrer ID passt. Ein beschädigter oder manipulierter Stand wird verworfen und ein neues Spiel gestartet. Karten werden ohne `innerHTML` gezeichnet (`face()` nutzt `textContent`).
- Ein gespeicherter Stand wird sofort angezeigt, ohne auf `solitaire-deals.json` zu warten. Nur ein neues Spiel braucht die Deck-Sammlung.

## Relevante Funktionen

| Funktion | Aufgabe |
| --- | --- |
| `cardButton()` | Karten-UI, native Drag- und Touch-Pointer-Events |
| `tapCard()` | Gemeinsame Tap-Logik für Maus und Touch: auswählen, verschieben, abwählen, Doppeltap |
| `onTap()` | Button-Aktion direkt im Touch-`pointerup`, der folgende Click wird ignoriert |
| `picked()` | Ermittelt die bewegte Karte bzw. Folge |
| `autoMove()` | Findet für Doppeltap ein erlaubtes Ziel |
| `canPlaceOnFoundation()` | Prüft die Grundstapel-Regel |
| `moveToTableau()` | Validiert und führt Tableau-Züge aus |
| `moveToFoundation()` | Validiert und führt Grundstapel-Züge aus |
| `animateCardMoves()` | Animiert Karten vom Ursprung zum Ziel |
| `revealAfterMove()` | Deckt nach der Bewegung eine freigelegte Karte auf |
| `autoFinishGame()` | Fertigspielen: Karten fliegen nacheinander auf die Zielstapel |
| `startHeap()` | Lässt nach dem Sieg alle Karten auf einen wilden Haufen fliegen |
