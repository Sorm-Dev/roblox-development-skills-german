---
name: roblox-standards
description: Verbindlicher Coding-Standard für alle Roblox-Projekte (Luau, .rbxl) — Architektur, Modulaufbau, Namensgebung, Kommentare, Remotes, Datenhaltung, Instanzverwaltung und Fallstricke der Testumgebung. Laden, bevor Roblox-Code geschrieben, geändert oder geprüft wird.
---

# Roblox Coding Standard

Verbindlicher Standard für alle Roblox-Projekte. Gilt für alle Rollen, die Code schreiben
oder prüfen (Game-Coder, UI-Entwickler, Code-Reviewer).

Der Reviewer zitiert Verstöße über die Regelnummer, z.B. `STANDARD: Verstoß gegen R14`.
Zum Nachschlagen dient der **Regelindex** am Ende des Dokuments — die Nummern laufen im
Text bewusst nicht der Reihe nach, weil eine einmal vergebene Nummer stabil bleiben muss.

**Status:** In Arbeit. Block 1–3 stehen und sind bindend, Block 4 (Verbote) steht noch
aus. Offene Punkte am Ende des Dokuments.

**Revision vom 9. August 2026** nach einer externen Prüfung: korrigiert wurden R2, R8,
R16, R59, R91 und die UI-Architektur (R112); neu sind R108–R115 und die Prüfkategorie
*Verfahren*. Geänderte Stellen tragen einen datierten Vermerk im Text. *(Die Statuszeile
selbst war ein Fund: sie meldete Block 3 als ausstehend, während er längst bindend im
Regelindex stand — ein R101-Fall im eigenen Haus.)*

---

## Grundentscheidungen

- **Kein Framework.** Reine ModuleScripts mit dem Init/Start-Muster aus R2. Kein Knit,
  kein Matter, kein Flamework. Begründung: kein Fremdcode, den der Reviewer nicht
  beurteilen kann, und das Muster ist in zehn Zeilen erklärt.
- **Alles im `.rbxl`.** Kein Rojo, keine Dateinamen-Konventionen wie `.server.luau` —
  in Roblox Studio ist der ClassName der Marker.
- **Code ist durchgängig englisch.** Bezeichner, Kommentare, Log-Ausgaben.

## Modulsorten

Das Dokument unterscheidet durchgehend vier Sorten. Viele Regeln gelten nur für eine
davon, deshalb steht die Einteilung hier und nicht verstreut.

| Sorte | Was | Wo | Wie erreicht |
| :--- | :--- | :--- | :--- |
| **Logik-Modul** | hält Zustand, hat Lebenszyklus (`Init`/`Start`) | `Services`, `Controllers` — UI-Logik ist ein Controller (R112) | injiziert (R31, R33) |
| **Hilfsmodul** | zustandslose Funktionen | `Shared.Util` | direkt requiret (R32) |
| **Datenmodul** | reine Tabellen, keine Logik | `Shared.Config`, `Shared.Data`, `Shared.Types`, `ServerUtils` | direkt requiret (R32) |
| **Einstiegspunkt** | lädt und startet die Logik-Module | `Main` auf Server und Client | — |

## Abweichungen

Regeln sind im Regelindex als **mechanisch**, **Ermessen** oder **Verfahren** eingestuft.

- **Mechanische Regeln sind hart.** Verstoß oder nicht, keine Begründung wird akzeptiert.
  Der Reviewer meldet sie als `STANDARD`.
- **Ermessensregeln dürfen begründet abweichen.** Die Begründung gehört in den Coder-
  Bericht unter „Bewusste Standard-Abweichungen". Der Reviewer meldet sie als `HINWEIS`,
  nicht als `STANDARD` — er kann widersprechen, aber nicht erzwingen.
- **Verfahrensregeln binden das Vorgehen, nicht den Code** — Messmethodik, Werkzeugkunde,
  Berichtspflichten. Ihre Einhaltung zeigt sich im Bericht und im Messaufbau, nicht in
  einer Codezeile; der Reviewer meldet Verstöße als `STANDARD` und zitiert sie gegen das
  Vorgehen. *(Kategorie eingeführt am 9. August 2026 — diese Regeln liefen zuvor als
  „mechanisch" und versprachen damit eine Prüfbarkeit am Code, die es nie gab.)*
- **Deckt keine Regel den Fall ab**, entscheidet der Coder nach der Begründung der
  nächstliegenden Regel und vermerkt es im Bericht. Regeln haben Begründungen, damit man
  sie auch dort anwenden kann, wo der Wortlaut nicht greift.

**In einem Bestand, der noch umgestellt wird, gilt der Standard für neuen und geänderten
Code sofort — auch wenn die umgebende Datei ihn noch verletzt.** Wer eine Datei anfasst,
schreibt seine eigenen Zeilen regelkonform, zieht den Rest aber nicht ungefragt nach.

Das gilt ausdrücklich auch für die Sprache (R16): neue Kommentare sind englisch, selbst in
einer Datei, deren Bestand noch deutsch ist. Ein gemischter Zwischenstand ist unschön, aber
er ist vorübergehend — eine Regel, die je nach Umgebung anders gilt, ist es nicht.

---

## Block 1 — Architektur

### DataModel-Struktur

```
ServerScriptService
  Main                    (Script)          ← einziger Server-Einstiegspunkt
  Services      (Folder)
    DataService           (ModuleScript)
    MatchService          (ModuleScript)
  ServerUtils   (Folder)  → Hilfs- und Datenmodule, die den Client nichts angehen (R32)

ReplicatedStorage
  Shared        (Folder)
    Config      (Folder)  → Stellschrauben
      Movement            (ModuleScript)
      Economy             (ModuleScript)
    Data        (Folder)  → Inhaltskataloge, thematisch getrennt
      Items               (ModuleScript)
      Levels              (ModuleScript)
    Types                 (ModuleScript)
    Util        (Folder)  → zustandslose Hilfsfunktionen
  Remotes       (Folder)  → zur Laufzeit vom Server befüllt

StarterPlayer
  StarterPlayerScripts
    Main                  (LocalScript)     ← einziger Client-Einstiegspunkt
    Controllers (Folder)
      InputController     (ModuleScript)
      MainMenuController  (ModuleScript)   ← UI-Logik ist ein Controller (R112)

StarterGui
  MainMenu      (ScreenGui)                 ← nur der GUI-Baum, kein Code (R112)
```

### Regeln

**R1 — Genau ein Einstiegspunkt pro Seite.**
Ein `Script` im Server, ein `LocalScript` im Client. Alles andere ist ModuleScript.
Keine Skripte, die irgendwo im Workspace hängen und zufällig loslaufen. Das macht die
Ladereihenfolge explizit und Fehler nachvollziehbar.

**R2 — Zweiphasiger Start.**
Jedes System ist ein ModuleScript mit fester Form: erst laufen alle `Init()` durch,
dann alle `Start()`. In `Init()` darf ein Modul nur seinen eigenen Zustand aufbauen und
**keine anderen Logik-Module aufrufen**; ab `Start()` sind alle da. Direkt requirete
Hilfs- und Datenmodule (R32) sind auch in `Init` zulässig — sie haben keinen Lebenszyklus,
auf den man warten müsste. *(Präzisiert am 9. August 2026: der alte Wortlaut „keine
anderen Module" verbot wörtlich auch harmlose Util-Aufrufe.)* Das löst
Reihenfolge-Abhängigkeiten ohne Framework und ohne `wait()`-Gefrickel.

**R3 — `Shared` ist zustandslos.**
Was in `ReplicatedStorage.Shared` liegt, hält keinen Zustand: Config sind Daten, Util
sind reine Funktionen, Types sind Typen. Zustand lebt ausschließlich in Services und
Controllers. Shared läuft auf beiden Seiten — Zustand dort bedeutet zwei divergierende
Kopien.

**R4 — Keine require-Ketten quer durch den Baum.**
Module holen sich einander über die Registry des Einstiegspunkts, nicht per
`require(game.ServerScriptService.Services.XY)` mitten aus einem anderen Modul heraus.
Sonst entstehen Zyklen, die erst zur Laufzeit knallen — und genau da beißt der
require-Cache. Die genaue Abgrenzung — was direkt requiret werden darf und was injiziert
wird — steht in R31–R37.

**R5 — Remotes an genau einer Stelle definiert.**
Ein Modul legt sie an und benennt sie, Server und Client greifen nur dort zu. Nie
irgendwo im Code ein `Instance.new("RemoteEvent")`.

**R6 — Ein Modul, eine Zuständigkeit.**
Wenn der Modulname ein "und" braucht, sind es zwei Module.

### Config vs. Data

**Config beantwortet "wie stark, wie schnell, wie lange".** Stellschrauben zum Drehen.
Flach, wenige Werte, wird beim Balancing angefasst. Bewegungsgeschwindigkeit,
Rundenlänge, Respawn-Zeit, Ökonomie-Raten, Feature-Flags.

**Data beantwortet "was gibt es".** Inhaltskataloge, thematisch getrennt, viele Einträge
nach gleichem Schema. Item-Liste, Level-Definitionen, Shop-Angebote, Achievements.

**Test für Zweifelsfälle:**
- *Wenn ich hier einen Eintrag ergänze — gibt es dann etwas Neues im Spiel?* → **Data**
- *Wenn ich hier eine Zahl ändere — fühlt sich etwas anders an?* → **Config**

Der Standardfall: der Schadenswert einer Waffe gehört **zum Waffen-Eintrag in Data**,
weil er eine Eigenschaft dieses Inhalts ist. Ein globaler Schadensmultiplikator gehört
in **Config**, weil er an allem gleichzeitig dreht.

**R7 — Reine Daten, keine Logik.**
Ein Config- oder Data-Modul gibt eine Tabelle zurück. Kein `require` auf Logik-Module,
kein Zustand, **keine Funktion in der zurückgegebenen Tabelle**.

**Erlaubt ist Ableitung auf Modulebene aus den eigenen Werten.** `Config.Board.SafeCount`
aus `CellCount` und `MineCount` auszurechnen ist Datenaufbereitung, keine Logik — und eine
abgeleitete Zahl von den Zahlen zu trennen, aus denen sie entsteht, zwingt jeden Leser,
zwei Module nebeneinanderzulegen. Ebenso erlaubt ist eine **lokale** Schreibabkürzung, die
nicht in der Rückgabe steht, etwa ein `_rgb` für hundert Farbliterale.

**Das tiefe Versiegeln kommt nach der Ableitung**, nie davor (R8).

**R8 — Zurückgegebene Tabellen werden tief versiegelt.**
Beide Ordner liegen in `Shared` und werden von vielen Stellen gelesen — mutiert ein
System die geteilte Tabelle, ändert sich still für alle anderen etwas. Eingefroren
knallt es sofort dort, wo jemand schreibt.

**`table.freeze` allein hält dieses Versprechen nicht: es friert flach.** Verschachtelte
Tabellen bleiben beschreibbar — `Items.Sword.Damage = 999` läuft gegen ein gefrorenes
`Items` klaglos durch, weil `Sword` selbst nie gefroren wurde. Und Verschachtelung ist
bei Katalogen der Normalfall, nicht die Ausnahme. Deshalb versiegelt ein `deepFreeze` in
`Shared.Util` rekursiv:

```lua
--- Freezes a table and every table reachable from it
local function deepFreeze(value: any): any
	if type(value) == "table" and not table.isfrozen(value) then
		table.freeze(value)

		for _, child in value do
			deepFreeze(child)
		end
	end

	return value
end
```

*(Korrigiert am 9. August 2026: die Regel verlangte `table.freeze` und versprach damit
einen Schutz, den nur die tiefe Variante liefert.)*

**Geprüft wird das Ergebnis, nicht das Werkzeug.** Wo `Shared.Util.deepFreeze` existiert,
versiegelt man damit. Wo es (noch) fehlt, ist **manuelles `table.freeze` je Ebene** — jede
verschachtelte Tabelle, jede Gruppe und die Wurzel — gleichwertig zulässig, solange nachweislich
nichts Erreichbares beschreibbar bleibt (`Items.Sword.Damage = 999` muss werfen). Der Reviewer
prüft die tiefe Unveränderlichkeit, nicht den Aufruf einer bestimmten Funktion. *(Ergänzt am
11. August 2026: die Data-Module dieses Projekts frieren durchgängig manuell tief — das erfüllt R8.)*

**R9 — Data-Einträge, auf die von außen verwiesen wird, haben stabile String-IDs als
Schlüssel**, keine Array-Indizes. Sonst verschieben sich Einträge beim Umsortieren und
gespeicherte Spielerdaten zeigen ins Leere.

*Eine Menge austauschbarer, namenloser Nutzlasten darf ein Array sein.* Das Kriterium ist
der Verweis, nicht die Form: Wenn nichts — kein anderer Code, keine gespeicherten
Spielerdaten — auf einen **einzelnen** Eintrag zeigt, kann sich beim Umsortieren auch
nichts verschieben. Zwanzig vorab geprüfte Spielbretter, aus denen zufällig eins gezogen
wird, sind kein Inhaltskatalog im Sinne von „Data beantwortet, was es gibt". Sie
durchzunummerieren erzwänge Namen, die niemand liest, und zerstörte `#liste` und die
Zufallswahl.

**R10 — Alles unter `Shared` ist für Exploiter lesbar.**
ReplicatedStorage wird komplett zum Client repliziert. Drop-Chancen, die niemand
ausrechnen können soll, Anti-Cheat-Schwellen, unveröffentlichte Inhalte — alles
Serverseitige gehört nach `ServerScriptService.ServerUtils` (R32).

**Und `ReplicatedStorage` ist nicht der einzige Weg zum Client.** Der Workspace repliziert
ebenso, samt Attributes (R81) — und Remotes tragen, was man ihnen mitgibt. Was der Spieler
nicht wissen darf, regelt deshalb eine eigene Regel: R108. *(Ergänzt am 9. August 2026.)*

### Modul-Registry

> Die Regelnummern R31–R37 laufen hier außer der Reihe. Der Reviewer zitiert
> Regelnummern, also müssen sie stabil bleiben — Umnummerieren wäre teurer als eine
> leicht unlogische Reihenfolge.

Die Abgrenzung, die R4 offenlässt:

> **Hilfs- und Datenmodule** werden **direkt requiret**.
> **Logik-Module** werden **nie requiret, sondern injiziert**.

Damit ist die Regel prüfbar: findet der Reviewer in einem Service ein `require` auf einen
anderen Service, ist das ein Fund. Alles andere im IMPORTS-Block ist erlaubt.

#### Das Muster im Modul

Der Loader übergibt allen Modulen die Registry an `Init`. Das Modul legt sich ab, was es
braucht — **abgelegt, nicht aufgerufen**, weil in `Init` laut R2 noch niemand fertig ist.

```lua
-------------
--- LOCAL ---
-------------

--- Modules provided by the loader
local _matchService
local _shopService


--------------
--- PUBLIC ---
--------------

local DataService = {}


--- Receives sibling modules from the loader
function DataService.Init(modules: { [string]: any })
	_matchService = modules.MatchService
	_shopService = modules.ShopService
end


--- Everything is loaded from here on
function DataService.Start()
	_matchService.RoundEnded:Connect(_saveAll)
end
```

Nebeneffekt für den Reviewer: die Abhängigkeiten eines Moduls stehen als `_`-Variablen-
block im LOCAL-Block unter einem `---`-Kommentar. Er sieht auf einen Blick, was das Modul
anfasst, ohne den Code zu lesen.

#### Der Loader

```lua
--- Loads and starts all server modules in a fixed two-phase order.


---------------
--- IMPORTS ---
---------------

local ServerScriptService = game:GetService("ServerScriptService")


-------------
--- LOCAL ---
-------------

--- Constants
local MODULE_FOLDER = ServerScriptService.Services


--- All loaded modules, keyed by their instance name
local _modules = {}


--- Requires every ModuleScript in the module folder
local function _loadModules()
	for _, child in MODULE_FOLDER:GetChildren() do
		if child:IsA("ModuleScript") then
			assert(_modules[child.Name] == nil, "Duplicate module name: " .. child.Name)
			_modules[child.Name] = require(child)
		end
	end
end


--- Runs one lifecycle phase across all loaded modules
local function _runPhase(phaseName: string)
	for _, module in _modules do
		local phase = module[phaseName]

		if type(phase) == "function" then
			phase(_modules)
		end
	end
end


------------
--- BOOT ---
------------

_loadModules()
_runPhase("Init")
_runPhase("Start")
```

Der Client-`Main` ist derselbe Loader auf `StarterPlayerScripts.Controllers`.
Server-Services tauchen nie in der Client-Registry auf — es sind zwei getrennte
Registries.

#### Regeln

**R31 — Nur der Einstiegspunkt requiret Logik-Module.**
Kein Service requiret einen anderen Service, kein Controller einen anderen Controller.

**R32 — Hilfs- und Datenmodule werden direkt requiret.**
Sie stehen ganz normal im IMPORTS-Block.

**Wo sie liegen.** Lesen **beide Seiten** sie, gehören sie nach `ReplicatedStorage.Shared`.
Liest sie **nur der Server**, gehören sie nach `ServerScriptService.ServerUtils`.

Das Kriterium für `ServerUtils` ist dreifach und muss **ganz** erfüllt sein:

1. **Kein Lebenszyklus** — weder `Init` noch `Start`.
2. **Wird direkt requiret**, nicht injiziert.
3. **Hat auf dem Client nichts verloren.**

Fehlt eines davon, gehört das Modul woanders hin: Ein Modul mit `Init` ist ein
Lebenszyklusmodul und gehört unter den Loader; ein Modul, das beide Seiten brauchen, gehört
nach `Shared`.

*Punkt 3 meint mehr als „enthält Geheimnisse".* Ein Werkzeug, mit dem Bretter gebaut werden,
enthält selbst keine Minen — es bekommt sie als Parameter. Trotzdem gehört es nicht auf den
Client: Wer es dort fände, wäre der Erste, der es benutzt, und er soll es nicht benutzen.

**Nicht unter den Loader**, auch wenn es dort funktionieren würde. Der Loader sammelt seine
Kinder als Lebenszyklusmodule ein und stellt sie in die Registry. Ein Hilfsmodul bekäme dort
einen Eintrag, den niemand injizieren soll — genau die Vermischung, die die Grenze zwischen
R31 und R32 verhindert. Das ist kein R37-Fall: R37 verbietet **Unterordner im
Loader-Ordner**, sie verlangt nicht, dass jedes Servermodul ein Kind des Loaders ist.

**R33 — Abhängigkeiten kommen über `Init(modules)`** und werden im LOCAL-Block unter
`--- Modules provided by the loader` abgelegt. In `Init` wird nur abgelegt, nie
aufgerufen.

Der Parameter wird annotiert: `Init(modules: { [string]: any })`. **R28 macht für
Lebenszyklusfunktionen keine Ausnahme** — sie formuliert genau eine, nämlich das leere
Klammerpaar bei fehlendem Rückgabewert. Und die Begründung der Regel greift hier besonders:
`modules` ist der eine Parameter, dessen Form man nicht raten kann.

**Der Preis der Injektion, ehrlich benannt:** `{ [string]: any }` macht jedes injizierte
Modul zu `any`. Am Aufrufort prüft dann **kein** Werkzeug mehr — R28-Annotationen wirken
nicht durch ein `any` hindurch, und ein Tippfehler im Feldnamen wird erst zur Laufzeit ein
`nil`-Fehler. Die Naht zwischen Modulen ist damit genau die Stelle mit der schwächsten
Prüfung im ganzen System. Wer das schließen will: R111. *(Ergänzt am 9. August 2026.)*

**R34 — Innerhalb einer Phase ist die Reihenfolge undefiniert.**
Kein Modul darf sich darauf verlassen, dass ein anderes in derselben Phase schon dran war.
Wer das braucht, braucht `Start` statt `Init`.

**R89 — In `Init` wird nichts verbunden und nichts gestartet.**
Keine Signalverbindung, kein `task.spawn`, kein Thread. **Yielden ist zulässig**, aber nur
mit Timeout nach R83.

Der Grund, warum beides zusammengehört: Ein Yield in `Init` ist genau dann gefährlich, wenn
während des Wartefensters ein Modul *lebendig* ist und in einen noch nicht initialisierten
Nachbarn greift. Verbindet und startet kein `Init` etwas, gibt es dieses Fenster nicht —
und dann ist Warten unbedenklich.

Umgekehrt ist `Start` nicht automatisch die sichere Wahl: Wer eine Ressource erst dort
bereitstellt, lässt jedes Modul auflaufen, dessen `Start` vorher dran ist (R34).
Bereitstellen gehört nach `Init`, Benutzen nach `Start`.

**Und deshalb gilt für `Start` das Gegenteil: dort wird nicht gewartet.** In `Init` ist ein
Yield unbedenklich, weil nach dieser Regel nichts lebt. In `Start` ist genau das Gegenteil
der Fall — wer schon gestartet ist, hat verbunden, und ein Wartefenster in `Start` ist ein
Fenster, in dem lebendige Module in ein halbfertiges Modul greifen können.

Wer also warten muss, wartet in `Init`. Ein `WaitForChild` nach `Start` zu verschieben,
weil `Init` „sauber bleiben" soll, verschiebt das Problem von der harmlosen in die
gefährliche Phase.

**Zwei Folgen des Yieldens in `Init`, die man wissen muss:** Der Loader ruft die
`Init`-Funktionen nacheinander auf — jede Wartezeit verzögert alle Module, die danach
drankommen. Wartezeiten in `Init` bleiben deshalb kurz. Und läuft ein Timeout ab, wird
**laut** gescheitert (`error`), nicht mit `nil` weitergearbeitet: R36 gilt sinngemäß, ein
Modul ohne seine Voraussetzung ist ein Startfehler und kein Sonderfall. Zu wissen dabei:
`WaitForChild` **mit** Timeout warnt — anders als die Variante ohne — nicht von selbst;
das laute Scheitern ist Aufgabe des Aufrufers. *(Ergänzt am 9. August 2026.)*

**R35 — `Init` und `Start` sind optional.**
Ein Modul ohne Lebenszyklus definiert sie einfach nicht.

**R36 — Der Loader fängt keine Fehler ab.**
Ein Modul, das beim Start scheitert, soll laut und sofort scheitern. Ein still
übersprungenes Modul erzeugt Folgefehler an ganz anderer Stelle, und man sucht sie an der
falschen.

**R37 — Der Modulordner ist flach.**
Der Loader geht genau eine Ebene tief. Verschachtelte Ordner werden nicht geladen. Grund:
die Registry geht über den Instanznamen — zwei gleichnamige Module in verschiedenen
Unterordnern würden sich still überschreiben.

Die Flachheit allein verhindert die Kollision übrigens nicht: Roblox erlaubt gleichnamige
Geschwister. Deshalb prüft der Loader beim Einsammeln auf Duplikate — die `assert`-Zeile
im Referenz-Loader macht aus dem stillen Überschreiben einen lauten Startfehler (R36).
*(Ergänzt am 9. August 2026.)*

> **Bewusst nicht gemacht:** keine `Dependencies`-Deklaration pro Modul, die der Loader
> validiert. Ein Tippfehler (`modules.MatchServic`) wird dadurch still `nil` und knallt
> erst an der Aufrufstelle. Der Preis einer Deklaration wäre, jede Abhängigkeit zweimal zu
> pflegen — die beiden Listen driften auseinander, und es ist der erste Schritt Richtung
> Framework. Der Fehler ist beim Testen sofort sichtbar.

**R111 — Wer die `any`-Naht schließen will, pflegt je Seite einen Registry-Typ an einer
Stelle.** *(Ermessen, ergänzt am 9. August 2026.)*

Der Ausweg, der mit dem Kasten oben verträglich ist: **ein** Typ pro Seite, ein Feld pro
Modul, gepflegt an genau einer Stelle — serverseitig in `ServerUtils`, clientseitig in
`Shared.Types`. Die Felder beziehen ihre Typen über `typeof(require(...))`; das läuft nur
im Typprüfer, nicht zur Laufzeit, und kollidiert deshalb nicht mit R31. Module annotieren
dann `Init(modules: Types.ServerRegistry)` statt `{ [string]: any }`.

Der Preis bleibt real — ein neues Modul heißt zwei Zeilen statt einer, und die zweite kann
vergessen werden. Deshalb Ermessen. Wer ihn zahlt, bekommt den Tippfehler
`_matchService.RondEnded` zur Analysezeit statt als stilles Laufzeit-`nil`.

### UI-Logik

*Ergänzt am 9. August 2026.* Bis dahin erklärte der Standard UI-Logik zum injizierten
Logik-Modul, sah sie aber unter `StarterGui` vor — wo der Client-Loader nie hinschaut.
Jede Auflösung dieses Widerspruchs verletzte eine Regel: requiret ein Controller das
UI-Modul direkt, fällt R31; lädt es niemand, läuft es nie. Dazu kommt, was `StarterGui`
technisch ist: eine **Vorlage**, die pro Spieler nach `PlayerGui` geklont wird. Ein
ModuleScript unter `StarterGui` und sein Klon im `PlayerGui` sind zwei verschiedene
Instanzen mit getrennten require-Caches — welches davon „das Modul" ist, hängt davon ab,
wer zuerst requiret. Das ist dieselbe Fehlerklasse wie R88, nur im Produktivcode.

**R112 — Unter `StarterGui` liegt kein Code. UI-Logik ist ein Controller.**
`StarterGui` enthält ausschließlich den GUI-Baum: ScreenGuis, Frames, Buttons. Die Logik
dazu ist ein gewöhnlicher Controller unter `StarterPlayerScripts.Controllers`
(`MainMenuController`) — er läuft über Loader, Registry und Lebenszyklus wie jeder andere
und holt sich seinen GUI-Baum in `Init` über
`Players.LocalPlayer.PlayerGui:WaitForChild(...)` mit Timeout (R83, R89).

**R113 — Über `ResetOnSpawn` wird pro ScreenGui entschieden, und die Entscheidung steht
im Controller.**
Der Engine-Default ist `true`: beim Respawn ersetzt ein frischer Klon der Vorlage das GUI,
und jede Referenz des Controllers zeigt auf den toten alten. Für Controller-verwaltete
GUIs ist `ResetOnSpawn = false` der Normalfall. Wo das Zurücksetzen gewollt ist, verbindet
der Controller `CharacterAdded` — mit Bestands-Nachzug nach R110 — und holt sich den Baum
neu. Eine der beiden Antworten muss
im Code erkennbar getroffen sein — ein Default, der zufällig noch nicht wehgetan hat, ist
keine Entscheidung.

---

## Block 2 — Skript-Aufbau

### Der Grundaufbau

**Der Kopf** ist ein kurzer englischer Einzeiler, der erklärt, wofür das Skript zuständig
ist. Kein Mehrzeiler.

**Trägt das Modul eine Modus-Direktive, steht sie darüber — in Zeile 1.**

```lua
--!strict
--- Owns mine placement, adjacency counts and per-cell state.
```

Grund: `--!`-Direktiven werden vom Parser im Dateikopf gelesen, und ob sie das auch hinter
einem vorangestellten Kommentar noch tun, ist eine Eigenschaft der Engine, die sich ändern
kann. Greift die Direktive nicht, **steht sie da und tut nichts** — man glaubt, Typprüfung
zu haben, und hat keine. Ein stiller Ausfall, wie ihn R94 beschreibt.

Die Direktive in Zeile 1 zu setzen macht die Frage gegenstandslos, statt sie zu
beantworten. Ohne Direktive beginnt die Datei unverändert mit dem Einzeiler.

Darunter folgen Banner-Blöcke. Der Code wird in diese Blöcke einsortiert:

```lua
---------------
--- IMPORTS ---
---------------


-------------
--- TYPES ---
-------------


-------------
--- LOCAL ---
-------------


--------------
--- PUBLIC ---
--------------


------------
--- BOOT ---
------------
```

Die Strichzeilen sind exakt so lang wie die mittlere Zeile.

Der TYPES-Block ist bedingt — er steht nur da, wenn das Modul tatsächlich Typen
definiert. Regeln dazu in R38 und R39.

**R11 — IMPORTS**
Requires und `game:GetService()`. Sonst nichts.

**R12 — LOCAL**
Lokaler Code — Variablen und Funktionen, die nur innerhalb dieses Moduls verfügbar sind.

Zulässig ist hier außerdem eine **ausführende Prüfanweisung**, die eine Invariante beim
Laden sichert — etwa ein `assert` auf einen Config-Wert, ohne den das Modul nicht sinnvoll
arbeiten kann. Sie steht direkt unter dem Wert, den sie absichert. Ein BOOT-Block ist dafür
nach R14 nicht zulässig, weil der den Einstiegspunkten vorbehalten ist.

**R13 — PUBLIC**
Alles, worauf von außerhalb des Moduls zugegriffen werden kann.

**R14 — BOOT**
Nur in Einstiegspunkten (Moduleloadern). Enthält den ausführenden Startcode, damit klar
ist, wo die Ausführung beginnt.

**R15 — Nicht jedes Modul hat alle Blöcke.** (Sorten siehe „Modulsorten")
- **Logik-Module und Hilfsmodule:** IMPORTS, LOCAL, PUBLIC — plus TYPES, wenn Typen
  definiert werden
- **Einstiegspunkte:** IMPORTS, LOCAL, BOOT — kein PUBLIC
- **Datenmodule:** keine Blockstruktur, sie geben nur eine Tabelle zurück

**Ein Block entfällt, wenn er leer bliebe.** Ein Modul, das nichts importiert, bekommt
keinen IMPORTS-Banner; eines ohne modulinternen Code keinen LOCAL-Banner. Ein Banner ohne
Inhalt erklärt nichts und kostet nur Zeilen.

**R40 — Feste Reihenfolge innerhalb von LOCAL.**
Konstanten → injizierte Module → Zustandsvariablen → Funktionen. Eine Abweichung sieht der
Reviewer beim Überfliegen, ohne den Code zu verstehen. Lua erzwingt es ohnehin halb: eine
`local function` ist erst ab ihrer Definitionszeile sichtbar, Hilfsfunktionen müssen also
vor ihren Aufrufern stehen.

Der Preis: in einem langen Modul stehen eine Variable und die Funktion, die sie benutzt,
weit auseinander. Das ist dann aber ein Hinweis darauf, dass das Modul gegen R6 verstößt.

**R84 — Die Modultabelle steht direkt unter dem PUBLIC-Banner**, vor allem anderen im
Block. `local DataService = {}` ist die erste Zeile nach dem Banner — danach kommen
öffentliche Felder, dann öffentliche Funktionen.

**R94 — Zustand, der eine laufende Interaktion überdauern muss, liegt nie in einer
schwachen Tabelle.**

`__mode = "k"` übergibt die Lebensdauer der eigenen Buchführung an den Garbage Collector.
Dessen Zeitpunkt steht in keinem Vertrag, und ein von der Engine gereichtes Objekt — ein
`InputObject`, ein Lua-seitiger Wrapper eines C++-Objekts — kann seine Lua-Referenz
verlieren, **während die Interaktion noch läuft**.

Der Ausfall ist still und zeitabhängig: er wirft nicht, er leckt nicht, **er schaltet eine
Prüfung ab**. Und weil er erst nach einer Verzögerung eintritt, sehen kurze Interaktionen
weiterhin richtig aus.

> **Der Anlass:** Die Tipp-gegen-Wisch-Erkennung hielt die Startposition eines Fingers in
> einer schwachen Tabelle. Der Kollektor holte den Schlüssel nach etwa 300 ms — während
> der Finger noch unten lag. Ab da maß die Abstandsprüfung nichts mehr. Drei statische
> Reviews hielten den Pfad für korrekt, zweimal ausdrücklich als „besonders genau geprüft"
> ausgewiesen. Gefunden hat es der erste Testlauf auf einem Gerät.

Wo eine Tabelle nicht unbegrenzt wachsen darf, wird sie **ausdrücklich** beschränkt: nach
Alter, nach Anzahl, oder über ein Ereignis, das garantiert kommt. Eine ausdrückliche
Schranke kann man lesen, prüfen und testen — den Kollektor nicht.

**R92 — Ein lokaler Handler ruft nie die eigene Modultabelle.**

R84 setzt `local Modul = {}` unter den PUBLIC-Banner, R40 setzt die lokalen Funktionen ans
Ende des LOCAL-Blocks. Damit steht die Modultabelle **hinter** jeder lokalen Funktion. Ein
`Modul.Etwas()` in einer lokalen Funktion löst den Namen nicht als Upvalue auf, sondern als
**globales `nil`** — und wirft erst zur Laufzeit, beim ersten Auslösen.

Braucht eine lokale Funktion Verhalten, das auch öffentlich sein soll, steht dieses
Verhalten in einer **lokalen** Funktion, und die öffentliche delegiert dorthin:

```lua
local function _toggleFlagMode()
	-- ...
end


function InputController.ToggleFlagMode()
	_toggleFlagMode()
end
```

Umgekehrt ist der Zugriff unbedenklich: eine öffentliche Funktion steht immer hinter der
Modultabelle und darf `Modul.Anderes()` rufen.

> Diese Falle ist eine Folge zweier eigener Regeln. Kein Werkzeug prüft sie, und sie fällt
> nicht beim Schreiben auf, sondern beim ersten Tastendruck im Betrieb.

### Namensgebung

**R16 — Der gesamte Code ist englisch — einschließlich der Instanznamen im DataModel.**
Ein `HauptMenue` im Explorer ist derselbe Verstoß wie ein `_spielerAnzahl` im Code: beide
tauchen in Pfaden, Registry-Schlüsseln und Berichten auf. *(Präzisiert am 9. August 2026 —
das Strukturdiagramm dieses Standards war bis dahin selbst das Gegenbeispiel.)*

**R17 — Der Unterstrich markiert den Scope, nicht die Sichtbarkeit.**

| Form | Bedeutung |
| :--- | :--- |
| `_playerCount` | Modul-Scope, modulintern |
| `PlayerCount` / `Module.GetEnemies()` | Modul-Scope, öffentlich |
| `playerCount` | Funktions-Scope |

Innerhalb einer Funktion ist ohnehin alles lokal — dort markiert der Unterstrich nichts.
Das deckt Schleifenzähler, `k`/`v`, temporäre Werte und `_` als bewusst ignorierten
Rückgabewert mit ab, ohne Ausnahmeliste.

**R18 — Konstanten sind die Ausnahme.**
`SCREAMING_SNAKE_CASE`, ohne Unterstrich, unter einem `--- Constants`-Kommentar oben im
LOCAL-Block.

**R19 — Namen geben auf den ersten Blick ein klares Bild.**
Keine kryptischen Bezeichner, keine Abkürzungen, die man kennen muss.

Ein-Zeichen-Namen sind nur im Funktions-Scope erlaubt und nur da, wo die Bedeutung
unstrittig ist: Koordinaten (`x`, `y`, `z`), Schleifenzähler (`i`), Schlüssel-Wert-Paare
(`k`, `v`) und `_` für einen bewusst ignorierten Rückgabewert. Auf Modulebene nie.

**R20 — Module, Services und Controllers: PascalCase, Substantiv, kein Präfix.**
`DataService`, nicht `MDataService` oder `dataService`.

**R85 — Booleans beginnen mit `is`, `has` oder `can`.**
`isAlive`, `hasKey`, `canJump`. Ein Boolean ohne eines dieser Präfixe ist ein Fund — man
soll an der Abfragestelle sehen, dass ein Wahrheitswert geprüft wird und nicht ein Objekt
auf Existenz.

### Kommentare

**R21 — Kommentare sind englisch und knapp.**

**R22 — Ein Kommentar über einer Funktion wird mit `---` eingeleitet**, direkt über der
Funktion. **Er ist Pflicht, wo es etwas zu begründen gibt — und nur dort.**

Was ein Kommentar verdient: eine Zahl, die nicht selbsterklärend ist; eine bewusste
Abweichung vom Naheliegenden; eine Falle für den Nächsten; eine Reihenfolge, die zwingend
ist. Was keinen verdient: eine Funktion, deren Name die Antwort schon gibt.

*Die Regel hieß bis zum 7. August 2026 „über **jeder** Funktion".* Das war ein Fehler mit
messbarem Preis: Wer kommentieren **muss**, schreibt auch dort etwas, wo nichts zu sagen
ist — und ein Agent schreibt im Zweifel zu viel. Zusammen mit R25 („erklär das Warum")
entstand Druck, ein Warum zu *erfinden*, wo keines war. Ergebnis waren Funktionen mit mehr
Kommentar als Code, und jeder Leser danach zahlt dafür.

**R23 — Über einem Block thematisch zusammengehöriger Variablen steht ein
`---`-Kommentar**, der in einem Satz sagt, was das für Variablen sind. Hier bleibt die
Pflicht: Ein Block ohne Überschrift ist beim Überfliegen nicht als Block erkennbar.

**R24 — Innerhalb von Funktionen stehen keine Kommentare.**
Wer eine Funktion nicht ohne innere Kommentare erklären kann, hat eine Funktion, die zu
viel tut.

**R25 — Kommentare erklären *warum*, nicht *was*.**
`-- increases i by 1` ist ein Fund. `-- recalculated on the server because the client
can tamper with this value` nicht.

*Und der Umkehrschluss, seit dem 7. August 2026 ausdrücklich:* Gibt es kein Warum, gibt es
keinen Kommentar. R25 ist keine Aufforderung, für jedes Was ein Warum zu suchen.

**R107 — Eine Begründung steht an einer Stelle, nicht an fünf.**

Braucht eine Entscheidung mehr als ein paar Zeilen Erklärung, gehört sie **an den einen Ort,
auf den die anderen zeigen** — nicht in jede Datei, die sie betrifft. Die übrigen Stellen
nennen die Aussage in einem Halbsatz und verweisen namentlich dorthin.

*Warum das keine Stilfrage ist:* Eine fünffach kopierte Begründung ist fünffach zu pflegen
und veraltet vierfach unbemerkt. Genau das ist hier passiert — eine Aussage über
`CanQuery` stand an sechs Stellen als glatte Tatsache, war falsch, und hat zwei Monate lang
überlebt, weil kein einzelner Kommentar sich als *die* Quelle verstand. Ein Ort lässt sich
korrigieren; sechs lassen sich nur finden.

Das gilt auch für Messwerte (siehe R106): Die Messung mit ihrer Bedingung steht **einmal**,
mit Datum. Wer sie zitiert, zitiert die Aussage und nennt die Quelle.

**R86 — Keine auskommentierten Code-Leichen.**
Weg damit. Der Rückweg ist Roblox' Versionshistorie oder die gespeicherte `.rbxl` — dafür
braucht es keine Grabsteine im Code.

**R87 — Ein `TODO` trägt ein Datum.**
`-- TODO(2026-08-01): ...`. Ein `TODO` ohne Datum ist ein Fund. Mit Datum sieht man beim
Lesen sofort, ob es von letzter Woche stammt oder seit einem halben Jahr niemanden
gestört hat.

**R103 — Text, den ein Spieler im Spiel sieht, ist englisch.**

R21 regelt die Sprache im Code. Diese Regel regelt die Sprache **auf dem Bildschirm**:
Knopfbeschriftungen, Hinweise, Meldungen, Namen von Gegenständen — alles, was ein Spieler
liest.

*Warum nicht die Sprache des Entwicklers:* Roblox' Publikum ist von Anfang an international,
und ein Spiel, das mit deutschen Beschriftungen anfängt, hat die Wahl zwischen einer
Umstellung später oder gar keiner. Englisch ist außerdem die Sprache, in der der übrige
Code steht — eine zweite Sprache im selben Modul erzwingt bei jeder Zeile die Frage, welche
gerade gilt.

*Ausgenommen ist, was ohnehin sprachlos ist:* ein Tastenbuchstabe, eine Zahl, ein
Eigenname.

*Prüfung: mechanisch.*

**R101 — Ein Kommentar, der behauptet, was ein *anderes* Modul tut, ist die Kopie eines
Vertrags. Wer den Vertrag ändert, ändert die Kopie mit — oder meldet sie.**

Die anderen Kommentarregeln schützen gegen Kommentare, die **geschrumpft** (R86),
**undatiert** (R87) oder **inhaltsleer** (R25) sind. Diese hier schützt gegen einen
Kommentar, der unwahr wird, **ohne dass jemand ihn anfasst**.

*Der reale Fall:* Ein Modul beschrieb seine eigene Vorprüfung mit „runs the same rules the
server runs — target present, in range, cell hidden, rate limit free". Ein Lauf fügte
serverseitig eine **fünfte** Regel hinzu. Der Kommentar stand danach unverändert da und war
falsch — in einer Datei, die der Lauf nie geöffnet hatte. Ein Diff zeigt so etwas nicht,
weil sich nichts geändert hat.

*Richtiges Vorgehen:* Wer eine Prüfkette, eine Signatur oder eine Zusicherung erweitert,
sucht nach Stellen, die sie **beschreiben**, nicht nur nach Stellen, die sie **aufrufen**.
Und wer eine solche Beschreibung schreibt, macht sie auffindbar: Der Name des Vertragsgebers
gehört hinein, damit die Suche sie findet.

Die Alternative zum Nachziehen ist immer zulässig: **den Kommentar melden statt ihn zu
reparieren.** Ein gemeldeter falscher Kommentar ist ein Auftrag, ein stiller ist eine Falle.

*Prüfung: Ermessen.*

### Leerzeilen

**R26 — Zwischen Funktionen stehen immer zwei Leerzeilen.**

**R27 — Thematisch zusammengehörige Variablen dürfen ohne Leerzeile untereinander
stehen.** Endet der Block, folgen zwei Leerzeilen.

**R41 — Um einen Block-Kommentar stehen zwei Leerzeilen darüber und eine darunter.**
Gilt auch für den ersten Block nach dem Kopf.

**R42 — Innerhalb eines Funktionsrumpfs steht höchstens eine Leerzeile.**
Die Zweier-Regel aus R26 und R27 gilt nur auf Modulebene.

### Formatierung

**R43 — Einrückung mit Tabs.**
Studio fügt standardmäßig Tabs ein, und im `.rbxl` läuft kein Formatter, der das
nachträglich geradezieht. Nebeneffekt: jeder kann seine Anzeigebreite selbst wählen.

**R44 — Maximale Zeilenlänge 120 Zeichen.**
Lange Instanzpfade und Typannotationen brauchen in Roblox-Code mehr Luft als in anderen
Sprachen.

**R45 — Doppelte Anführungszeichen.**
Die Roblox-API und Studios Autovervollständigung verwenden doppelte. Alles andere erzeugt
laufend Mischformen.

**R46 — Mehrzeilige Tabellen bekommen ein abschließendes Komma beim letzten Eintrag.**
Ein neuer Eintrag ändert dann genau eine Zeile statt zwei — im Skript-Spiegel sieht der
Reviewer damit nur die echte Änderung.

```lua
return {
	Coins = 0,
	Level = 1,
}
```

**R47 — Keine Semikolons, eine Anweisung pro Zeile.**

**R48 — Kein einziges globales Symbol.**
Alles wird `local` deklariert. Ein vergessenes `local` erzeugt eine Variable, die über
Modulgrenzen hinweg sichtbar ist und still von anderem Code überschrieben werden kann.

### Typen

**R28 — Öffentliche Funktionen haben annotierte Parameter und einen expliziten
Rückgabetyp.**
Die Annotation ist Dokumentation, die nicht veralten kann wie ein Kommentar — und sie
wird auch ohne `--!strict` geprüft.

Funktionen **ohne** Rückgabewert brauchen keine Annotation: `function X.Start()` statt
`function X.Start(): ()`. Ein leeres Klammerpaar erklärt nichts und stünde sonst in jedem
Modul mehrfach herum.

**R29 — `--!strict` ist nicht vorgeschrieben.**
Erlaubt und erwünscht in **Hilfs- und Datenmodulen**, weil die das DataModel nicht
anfassen und dort kein Cast-Rauschen entsteht. In **Logik-Modulen** und
**Einstiegspunkten** entscheidet der Coder.

Wo eine Direktive gesetzt wird, steht sie in **Zeile 1**, über dem Kopf — siehe „Der
Grundaufbau".

**R30 — Ein Typ, den mehr als ein Modul braucht, wandert nach `Shared.Types`**
und wird `export type`. Typen, die nur ein Modul benutzt, bleiben lokal.

**R38 — Definiert ein Modul Typen, stehen sie in einem `--- TYPES ---`-Block direkt nach
IMPORTS** — lokale und exportierte gemeinsam. Definiert es keine, entfällt der Block.
Typen sind Deklaration, kein Code; sie gehören nach oben und nicht in die Sortierung nach
Sichtbarkeit.

**R39 — Typnamen sind immer PascalCase, ohne Unterstrich.**
Die Ausnahme von R17: die Sichtbarkeit eines Typs markiert das Schlüsselwort `export`,
nicht der Name. `_Profile` ist ein Fund.

> **Konsequenz für den Reviewer:** Ohne strict prüft niemand automatisch, ob die
> Annotationen noch stimmen. "Passt die Signatur noch zu dem, was die Funktion tut?"
> gehört fest in die Review-Checkliste.

---

## Block 3 — Roblox-Praxis

> **Stand: 1. August 2026.** Dieser Block altert schneller als der Rest, weil er auf
> konkrete Engine-APIs zeigt. Jede Modernitätsregel nennt die aktuelle API **und** das,
> was sie ersetzt — damit der Reviewer das Alte per Grep findet.

### Grundsatzentscheidung: kein Server-Authority-Modell

Roblox' Server-Authority-Modell (`Workspace.AuthorityMode = Server`) mit Client-Prediction
und Rollback wird in diesen Projekten **nicht** verwendet. Stand 1. August 2026.

Die Property schaltet automatisch fünf weitere mit — `NextGenerationReplication`,
`PlayerScriptsUseInputActionSystem`, `SignalBehavior = Deferred`, `UseFixedSimulation`,
`StreamingEnabled` — und verlangt ein anderes Programmiermodell:

- Kernlogik in Funktionen, die über `RunService:BindToSimulation()` gebunden werden, in
  einem ModuleScript, das auf Client **und** Server gleichermaßen läuft
- Die Simulation muss deterministisch sein, damit der Client nach einer Fehlvorhersage
  identisch nachsimulieren kann
- Input ausschließlich über das Input Action System

Das kollidiert mit Block 1: unsere Trennung in Services auf dem Server und Controllers auf
dem Client mit zwei getrennten Registries setzt das klassische Modell voraus. Server
Authority will geteilte Simulationsmodule. **Ein Spiel mit Server Authority bräuchte einen
eigenen Architekturstandard, keinen erweiterten** — deshalb ist das hier keine offene
Option, sondern eine Abgrenzung.

Die erzwungenen Schalter sind für sich schon Umbauten: `SignalBehavior = Deferred` ändert
das Zeitverhalten von Events, `StreamingEnabled` ändert, was auf dem Client überhaupt
existiert.

> **Zusammenhang mit R49–R57:** Server Authority erzwingt IAS. Die Entscheidung gegen IAS
> und die Entscheidung gegen Server Authority sind dieselbe Entscheidung — wer eine davon
> neu bewertet, bewertet beide.

**Stand in einem realen Projekt, geprüft am 1. August 2026:**
`Workspace.AuthorityMode = Automatic` — das Server-Authority-Modell ist **nicht** aktiv,
die Abgrenzung oben hält.

`Workspace.PlayerScriptsUseInputActionSystem` ist dagegen **Enabled**, unabhängig davon.
Folge: Roblox' eigene Charakter- und Kamerasteuerung läuft bereits über das Input Action
System, und deshalb liegt in `Players.<name>.PlayerScripts` **kein `PlayerModule`** —
dort stehen nur `RbxCharacterSounds` und der `ClientLoader`.

Das ist kein Fehler und ändert an R49–R57 nichts: die Property migriert ausschließlich
Roblox' eigene Skripte, eigene `ContextActionService`-Bindungen laufen unverändert. Wer
allerdings Bewegung oder Kamera anpassen will, arbeitet in diesem Projekt mit IAS und
nicht mit dem klassischen `PlayerModule` — und wer einen Test zur Kamera- oder
Charaktersteuerung schreibt, sollte wissen, dass der Standard-Steuerungsstack hier anders
aussieht als in einem frischen Place.

### Teil 1 — Input

Aufbau und Referenzcode des `InputController` stehen in `PROJEKT-AUFSETZEN.md`.

**R49 — Nur der `InputController` berührt eine Input-API.**
Kein anderes Modul ruft `ContextActionService`, `UserInputService` oder
`Player:GetMouse()` auf. Alle anderen Module reagieren auf Signale des `InputController`.
Er liegt unter `StarterPlayerScripts.Controllers.InputController`.

Damit ist ein späterer Wechsel des Input-Systems ein einziges Modul statt ein Streifzug
durch die Codebasis. Unabhängig davon bündelt es die Eingabebehandlung an einer Stelle.

**R50 — `ContextActionService` für Aktionen, `UserInputService` nur für Abfragen.**
CAS für alles, was eine Spielaktion auslöst. UIS ausschließlich für Geräteabfragen
(`GetPlatform`, `TouchEnabled`, `KeyboardEnabled`, `GamepadEnabled`), kontinuierliche
Zeigerposition und Thumbstick-Werte, die gepollt werden müssen.
**`Player:GetMouse()` ist verboten** — Legacy-Weg, den `UserInputService` ersetzt.

**R51 — Aktionsnamen sind Konstanten.**
Alle Namen im `--- Constants`-Block des Input-Controllers, nie als Stringliteral an der
Binde- oder Entbindestelle. Ein Tippfehler beim `UnbindAction` schlägt **nicht** fehl — er
tut nichts, und die Aktion bleibt für den Rest der Sitzung gebunden. Stumme Fehler sind
die teuersten.

**R52 — Jede Aktion deckt Tastatur/Maus, Gamepad und Touch ab.**
Gamepad-KeyCodes werden beim `BindAction` mit angegeben, über das
`createTouchButton`-Argument wird bewusst entschieden. Eine Aktion mit nur einer
Tastatur-KeyCode ist ein Fund. Mobile ist bei Roblox die größte Plattform.

**R53 — Handler geben ihr Ergebnis explizit zurück.**
`Enum.ContextActionResult.Sink` oder `.Pass`, kein Handler endet ohne Rückgabe. Ohne
Rückgabe gilt implizit `Pass` und die Eingabe läuft an tiefer gebundene Aktionen weiter —
ein Bug, der erst auffällt, wenn sich zwei Aktionen eine Taste teilen.

**R54 — Moduswechsel über Binden und Entbinden, nicht über Flags.**
Soll Gameplay-Input während eines Menüs ruhen, werden die Gameplay-Aktionen entbunden.
Kein `if _isMenuOpen then return end` in den Handlern — sonst verteilt sich die Moduslogik
über alle Handler und der eine vergessene wird zum Bug.

**R55 — Was gebunden wird, wird wieder entbunden.**
Gebundene Aktionen überleben den Kontext, in dem sie gebunden wurden. Zu jeder Bindung
gehört eine Entbindung an der Stelle, an der der Kontext endet.

**R56 — Signale gibt der Controller als `BindableEvent` aus, zur Laufzeit erzeugt.**
Das `BindableEvent` bleibt privat, öffentlich ist nur `.Event`. Damit kann kein fremdes
Modul das Signal auslösen. Kein Instanzbaum, kein Setup pro Projekt.

```lua
-------------
--- LOCAL ---
-------------

--- Bindables backing the public signals
local _jumpBindable = Instance.new("BindableEvent")


--------------
--- PUBLIC ---
--------------

local InputController = {}


--- Fires when the player triggers the jump action
InputController.Jump = _jumpBindable.Event
```

Die Referenz auf das `BindableEvent` muss gehalten werden — `Instance.new("BindableEvent").Event`
ohne lokale Variable verliert das Objekt.

**R57 — Über Signale werden keine Tabellen geschickt.**
`BindableEvent` kopiert Tabellen bei der Übergabe und verliert dabei Metatabellen. Sicher
sind Primitive, `Vector2`/`Vector3`, `CFrame` und Instanzen.

> **Notiz zum Input Action System.** Roblox baut mit IAS (`InputContext` → `InputAction` →
> `InputBinding`) den Nachfolger und migriert bereits die eigenen Charakterskripte darauf
> (`Workspace.PlayerScriptsUseInputActionSystem`). Am 1. August 2026 sprach zweierlei
> dagegen: der Studio-Editor ist Beta, und die Dokumentation lässt zentrale Fragen offen —
> wo ein `InputContext` zur Laufzeit hängen muss und wie `Priority` und `Sink` zwischen
> mehreren Contexts zusammenwirken. Für ein Agenten-Team wiegt dünne Dokumentation
> schwerer als für einen Menschen: der Coder rät, und der Reviewer liest dieselben
> lückenhaften Docs und kann es nicht widerlegen. **Neu bewerten, wenn IAS die Beta
> verlässt** — dank R49 ist die Umstellung dann ein Modul.

### Teil 2 — Remotes und Server-Autorität

#### Die Haltung, aus der die folgenden Regeln kommen

Ein Roblox-Client läuft auf einem fremden Rechner. Alles, was dort entschieden wird, kann
jemand anders entscheiden — nicht theoretisch, sondern mit frei verfügbaren Werkzeugen und
ohne Programmierkenntnisse.

Die Regeln unten sind die **Antworten** auf diese Lage. Sie sind nicht die Frage. Die Frage
stellt sich bei jedem neuen Feature neu:

> **Was geht hier über die Grenze, was könnte ein Client davon fälschen, und was hindert
> ihn? Und in der Gegenrichtung: was erfährt der Client dadurch, das der Spieler nicht
> wissen darf?**

Sie hat drei Ausgänge, und alle drei sind in Ordnung:

- Nichts geht über die Grenze — dann ist die Antwort „nichts", und das kostet zehn Sekunden.
- Etwas geht hinüber, und der Server prüft es ohnehin — dann sag, wo.
- Etwas geht hinüber, und niemand prüft es — dann hast du gerade den Fund gemacht, für den
  es diese Frage gibt.

Die Gegenrichtung — das **Erfahren** — hat dieselben drei Ausgänge und denselben teuren
Fall. Sie fehlte in dieser Frage bis zum 9. August 2026: gefragt war, was ein Client
fälschen könnte, nie, was er wissen würde. Bei einem Spiel, dessen Kern verborgene
Information ist, ist das die teurere Hälfte — die Antwort regelt R108.

Der teure Fall ist keiner davon. Der teure Fall ist, die Frage **nicht** zu stellen — weil
etwas, das nach Bedienbarkeit aussah, hinterher eine Berechtigung war. Ein Werkzeug im
Rucksack ist der Klassiker: Es fühlt sich an wie ein Knopf und ist beinahe ein Schlüssel.

*Diese Frage trägt bewusst keine Regelnummer.* Eine Nummer verspricht Prüfbarkeit, und ob
jemand sich etwas gefragt hat, kann niemand prüfen. Prüfbar ist erst die **Antwort** — und
die steht deshalb, für beide Richtungen der Frage, als Pflichtfeld im Bericht des Coders.

**R58 — Der Client meldet Absichten, nie Ergebnisse.**
`FireServer("Jump")`, nicht „ich habe 50 Schaden gemacht". Der Server rechnet das Ergebnis
selbst aus. Ein Client, der Ergebnisse liefern darf, ist ein Client, der sie erfindet.

**R59 — Jedes Argument wird serverseitig validiert, bevor irgendetwas passiert.**
Typ, Wertebereich, und ob dieser Spieler diese Aktion überhaupt ausführen darf. Exploiter
feuern jedes Remote mit beliebigen Argumenten zu jedem beliebigen Zeitpunkt.

Zur Typprüfung einer Zahl gehören die Werte, die `type(x) == "number"` bestehen und
trotzdem keine sind: `NaN` (prüfbar über `x ~= x`), `math.huge` und `-math.huge`. Zu einem
String gehört seine Längengrenze, zu einer Tabelle ihre Tiefe und Größe — ein Client kann
beliebig große Nutzlasten schicken.

*Präzisiert am 9. August 2026 — was hier mechanisch ist und was nicht:* **Dass** zu jedem
Argument eine Validierung existiert, ist mechanisch prüfbar und bleibt `STANDARD`. **Ob
sie ausreicht**, ist Ermessen — der Reviewer prüft sie inhaltlich gegen die Frage „was
würde ein feindlicher Wert hier anrichten" und meldet Zweifel als `HINWEIS`. Ein Bericht,
der R59 als bestanden führt, sagt damit nur, dass geprüft *wird* — nicht, dass richtig
geprüft wird.

**R60 — `RemoteFunction:InvokeClient()` ist verboten.**
Roblox warnt selbst davor, und die Gründe sind hart: Gibt der Client nichts zurück,
**yieldet der Server für immer**. Wirft der Client einen Fehler, wirft der Server ihn auch.
Trennt der Client währenddessen die Verbindung, wirft der Aufruf. Server→Client läuft
immer über `RemoteEvent:FireClient()`.

**R61 — `RemoteFunction` nur, wenn der Client wirklich auf eine Antwort warten muss.**
Sie blockiert den aufrufenden Thread. Im Zweifel zwei `RemoteEvent`s statt einer
`RemoteFunction`.

**R62 — `UnreliableRemoteEvent` nur für kontinuierliche, unkritische Daten.**
Keine Reihenfolge, keine Zustellgarantie. Nie für Treffererkennung, Käufe, Punktevergabe
oder irgendetwas, das den Spielzustand ändert. Positions- und Animationsupdates ja.

**R63 — Über die Grenze gehen nur einfache Daten.**
Keine gemischten Tabellen (numerische **und** String-Schlüssel), kein `nil` als Index oder
Wert, keine Funktionen (kommen als `nil` an), keine Metatabellen (gehen verloren), keine
Instanzen aus `ServerStorage` (kommen als `nil` an). Tabellen werden kopiert — die
Identität ist auf beiden Seiten eine andere.

**R64 — Jedes Remote hat serverseitig eine Drosselung.**
Die Engine-Grenze von rund 500 Anfragen pro Sekunde und Client ist **keine** Verteidigung —
ein Exploiter feuert 400 pro Sekunde völlig legal. Pro Remote ein Minimalintervall je
Spieler. Die Höhe des Intervalls ist Ermessenssache, dass es eines gibt, nicht.

Und eine Drosselung ist **keine** Atomarität: zwei Anfragen im erlaubten Abstand können
beide dieselbe Prüfung bestehen, bevor die erste wirkt — das regelt R109.

**R65 — Remote-Namen sagen die Richtung.**
Imperativ für Anfragen vom Client (`RequestPurchase`), Vergangenheitsform für Meldungen vom
Server (`PurchaseCompleted`). An der Aufrufstelle soll erkennbar sein, wer wen anspricht.

**R108 — Verborgener Spielzustand liegt bis zur Aufdeckung ausschließlich auf dem
Server.**
Was der Spieler nicht wissen darf, existiert auf dem Client nicht — nicht als Attribut an
einer Workspace-Instanz (Attributes replizieren, R81), nicht als vorab mitgeschicktes Feld
in einem Remote-Payload, nicht als unsichtbar geschaltetes GUI. Der Client bekommt Zustand
in dem Moment, in dem der Spieler ihn regulär erfahren dürfte, und keinen Frame früher.

R58 und R59 schützen die **Integrität** des Zustands, diese Regel seine
**Vertraulichkeit** — das sind verschiedene Fragen. Ein Spiel kann jede Fälschung abwehren
und trotzdem seine gesamte verborgene Information an jeden Client senden. Bei einem Spiel,
dessen Kern verborgene Information ist — wo die Minen liegen —, ist diese Regel die
wichtigere der beiden.

*Prüfweg für den Reviewer:* nicht von den Remotes aus denken, sondern von den Daten: wo
wohnt das Verborgene, und auf welchen Wegen verlässt etwas davon den Server? Jeder Weg
wird einzeln geprüft. *(Ergänzt am 9. August 2026.)*

**R109 — Zwischen serverseitiger Prüfung und ihrer Wirkung liegt kein Yield.**
Prüfen, dann yielden, dann wirken ist das Muster hinter Item-Duplikation: zwei Anfragen
desselben Spielers bestehen **beide** die Guthabensprüfung, bevor die erste abbucht. Die
Drosselung aus R64 verhindert das nicht — beide Anfragen können regelkonform langsam sein.

Prüfung und Wirkung stehen deshalb im selben synchronen Abschnitt. Wo ein Yield dazwischen
unvermeidbar ist (`MarketplaceService`, DataStore), schützt ein Wiedereintritts-Guard je
Spieler und Aktion: beim Eintritt setzen, laufende Anfragen abweisen, am Ende löschen —
auch im Fehlerfall, sonst sperrt ein einziger Fehler den Spieler für den Rest der Sitzung.
*(Ergänzt am 9. August 2026.)*

### Teil 3 — Datenhaltung

Aufbau und Referenzcode stehen in `PROJEKT-AUFSETZEN.md`.

**R66 — Spielerdaten laufen ausschließlich über ProfileStore.**
Kein direkter `DataStoreService`-Zugriff auf Spielerschlüssel, auch nicht „nur kurz zum
Nachschauen". Session-Locking von Hand zu bauen ist die einzige Fehlerklasse in diesem
Standard, bei der der Schaden beim Spieler landet: Item-Duplikation und verlorene
Spielstände.

**R67 — Nur der `DataService` spricht mit ProfileStore.**
Andere Module holen sich Daten über seine öffentliche API. Gleiches Muster wie R49 beim
Input.

**R68 — Jeder DataStore-Aufruf außerhalb von ProfileStore steckt in einem `pcall`.**
Es sind Netzwerkaufrufe, sie schlagen gelegentlich fehl.

**R69 — `UpdateAsync`, wenn der neue Wert vom alten abhängt.**
`SetAsync` nur beim bedingungslosen Überschreiben — es kann Daten inkonsistent machen, wenn
zwei Server denselben Schlüssel gleichzeitig setzen. Zu bedenken: `UpdateAsync` verbraucht
Lese- **und** Schreibbudget.

**R70 — Im `UpdateAsync`-Callback wird nicht geyieldet.**
Kein `task.wait()`, kein weiterer Async-Aufruf. Die Engine erlaubt es nicht.

**R71 — Gespeicherte Daten haben eine Schemaversion.**
Ein Profil, das mit Version 1 gespeichert wurde, wird irgendwann von Version 3 des Spiels
geladen. `Reconcile()` ergänzt nur fehlende Felder — für ein Feld, das seine Bedeutung
geändert hat, braucht es Versionsnummer und Migrationspfad.

**R72 — Nichts unbegrenzt Wachsendes speichern.**
Keine Logs, keine Verlaufslisten ohne Obergrenze. 4 MB pro Schlüssel klingt viel, bis eine
Liste jeden Spielzug anhängt — dann schlägt das Speichern irgendwann komplett fehl.

**R73 — Ranglisten laufen über `OrderedDataStore`, nicht über ProfileStore.**
Der Autor grenzt das ausdrücklich aus: ProfileStore ist nicht für Ranglisten oder globalen
Zustand gedacht.

**R116 — Kurzlebiger, server-übergreifend geteilter Zustand läuft über `MemoryStore`, nie über
ProfileStore.**
Matchmaking-Warteschlangen, ein server-übergreifender Live-Zustand, eine Drosselung über Server
hinweg — Daten, die schnell und von mehreren Servern gelesen und geschrieben werden und nicht
dauerhaft bleiben müssen. `MemoryStore` ist dafür gebaut (hoher Durchsatz, niedrige Latenz);
ProfileStore ist session-gelockt und pro-Spieler, ein DataStore zu langsam und zu eng budgetiert.

**`MemoryStore` ist ausdrücklich nicht dauerhaft:** Einträge tragen eine TTL (höchstens 45 Tage)
und werden unter Speicherdruck früher verworfen. Was überleben muss, gehört **zusätzlich** in
ProfileStore (pro Spieler) oder `OrderedDataStore` (Rangliste) — MemoryStore ist der schnelle,
geteilte Zwischenspeicher, nicht die Quelle der Wahrheit. *(Ergänzt am 10. August 2026.)*

**R117 — Entwicklungs- und Produktionsdaten trennt ein Umgebungs-Scope, gewählt per
`RunService:IsStudio()`.**
Jeder Store bekommt einen Scope, der in Studio `"dev"` und auf einem echten Server `"prod"` ist:

```lua
local ENVIRONMENT = RunService:IsStudio() and "dev" or "prod"
```

Damit schreibt ein Studio-Test **nie** in die veröffentlichten Daten — auch nicht bei aktiviertem
„Enable Studio Access to API Services", das sonst dieselben Live-Stores anspricht. Der Wert wird
**einmal** im `DataService` bestimmt (R67) und fließt von dort als Scope in **jeden** Store —
ProfileStore, `OrderedDataStore`, `MemoryStore`.

**Warum `IsStudio()` und kein `Config`-Flag:** Ein bewusst gesetztes Flag hängt am Nicht-Vergessen,
und ein beim Veröffentlichen falsch stehen gelassenes schriebe Produktion in den Dev-Scope.
`IsStudio()` entscheidet ohne Zutun. Der Preis: ein dedizierter Test-Server (Team Test) zählt als
Produktion — bewusst in Kauf genommen; wer die Trennung dort braucht, erweitert die eine Stelle.
*(Ergänzt am 10. August 2026.)*

**R74 — Externe Bibliotheken werden über `insert_asset` mit festgeschriebener Asset-ID
installiert. Kein Agent kopiert fremden Quellcode in ein Projekt.**
`multi_edit` legt ein Skript an, indem das Modell den kompletten Inhalt ausgibt — bei
tausend Zeilen fremdem Code ist das keine Kopie, sondern eine Abschrift, und ein stiller
Abschriftfehler ist vom Reviewer nicht auffindbar. Bei `insert_asset` kopiert die Engine.
Die verifizierten IDs stehen in `PROJEKT-AUFSETZEN.md`.

**Eine Asset-ID ist allerdings kein Content-Hash.** Der Creator kann das Asset unter
derselben ID aktualisieren; `insert_asset` holt die jeweils aktuelle Fassung. Nach dem
ersten Einfügen lebt die Kopie im `.rbxl` und ist damit eingefroren — aber jedes **neue**
Projekt zieht potenziell eine andere Version. Nach dem Einfügen wird deshalb die Version
gegen die in `PROJEKT-AUFSETZEN.md` notierte geprüft. *(Ergänzt am 9. August 2026.)*

> **Budgets im Blick behalten.** DataStore-Anfragen haben zwei Grenzen gleichzeitig, je
> Minute: pro Experience (Lesen `300 + CCU × 40`, Schreiben `300 + CCU × 20`) und pro
> Server (Lesen und Schreiben je `60 + Spieler × 40`). Überschrittene Anfragen landen in
> einer Warteschlange mit 30 Plätzen — darüber werden sie **verworfen**.

### Teil 4 — Instanzen und Aufräumen

Zwei Tatsachen aus den offiziellen Docs, aus denen sich der Rest ergibt:

- **Die Engine sammelt Events, die an eine Instanz gebunden sind, nie ein** — samt aller
  Werte, die im Callback referenziert werden. Eine aktive Verbindung ist für den Garbage
  Collector unsichtbar.
- **`Destroy()` trennt die Events der Instanz, räumt aber deine eigenen Tabellen nicht
  auf.** Ein Eintrag in `_profiles[player]` überlebt jedes Destroy.

Und der Fall, den Roblox ausdrücklich als „sehr erhebliche Speicherlecks" bezeichnet:
**Player-Objekte und Charakter-Modelle werden nach dem Verlassen nicht automatisch
zerstört.**

**R75 — Jede Verbindung, die nicht für die gesamte Laufzeit besteht, wird nachgehalten und
getrennt.**
Verbindungen, die bewusst dauerhaft sind, bekommen einen Kommentar, der das sagt — sonst
sieht der Reviewer keinen Unterschied zwischen „dauerhaft gewollt" und „vergessen".

**R76 — Beim Verlassen eines Spielers wird alles zu ihm Gehörende freigegeben.**
Verbindungen **und** Tabelleneinträge. Eine Tabelle, die pro Beitritt wächst und beim
Verlassen nicht schrumpft, ist ein Leck, das mit der Serverlaufzeit skaliert.

**R110 — Ein `Added`-Signal meldet nur die Zukunft. Wer es verbindet, zieht den Bestand
nach.**
Zwischen dem Entstehen eines Objekts und dem `Connect` in `Start` kann beliebig viel Zeit
liegen — das Signal für alles, was vorher da war, ist unwiederbringlich verpasst. Das
Muster ist immer dasselbe: erst verbinden, dann den Bestand durch **denselben** Handler
schicken.

- `Players.PlayerAdded` → danach `Players:GetPlayers()`. Im Studio-Playtest ist der
  Spieler praktisch immer schneller als der `Connect`, auf echten Servern bei schnellen
  Joins ebenso.
- `player.CharacterAdded` → danach `player.Character`, falls gesetzt. Auf dem Client
  existiert der Charakter beim Start der Controller fast immer schon — dort ist der
  Backfill der Normalfall und das Signal die Ausnahme.
- `CollectionService:GetInstanceAddedSignal(tag)` → danach
  `CollectionService:GetTagged(tag)`. Diese Falle stellt R82 selbst, weil er zu
  Tag-Signalen rät.
- `ChildAdded` → danach `GetChildren()`.

```lua
player.CharacterAdded:Connect(_onCharacter)

if player.Character then
	_onCharacter(player.Character)
end
```

Der Handler muss beide Wege vertragen: keine Annahme, dass das Objekt frisch ist. Ein
nachgezogener Charakter kann Sekunden alt sein, und seine Kinder brauchen auf beiden
Wegen `WaitForChild` mit Timeout (R83). Ein Modul, das ein `Added`-Signal verbindet und
den Bestand nicht nachzieht, ist ein Fund; das Referenzbeispiel zeigt die Form für
`PlayerAdded`. Dieser Fehler besteht jeden Test, bei dem der Tester nach dem Auslöser
startet — und trifft dann den ersten echten Fall.
*(Ergänzt am 9. August 2026, verallgemeinert von `PlayerAdded` auf alle `Added`-Signale
am selben Tag.)*

**R77 — Auf `Workspace.PlayerCharacterDestroyBehavior` wird sich nicht verlassen.**
Die Property bleibt auf ihrem Standardwert. Aufgeräumt wird immer selbst, nach R75 und
R76 — Verbindungen trennen, Tabelleneinträge löschen.

Begründung: die Engine nimmt uns die Arbeit bestenfalls teilweise ab, und Code, der von
einer Place-Einstellung abhängt, bricht still, sobald jemand ein neues Projekt ohne sie
aufsetzt. Manuelles Aufräumen ist in jedem Fall richtig, unabhängig davon, was die
Property gerade tut.

**R78 — `Instance:Destroy()`, nie `:Remove()`.**
`Remove()` nimmt die Instanz nur aus dem Baum, zerstört sie aber nicht — sie bleibt samt
Verbindungen am Leben.

**R79 — Nach `Destroy()` wird die eigene Referenz auf `nil` gesetzt.**

**R91 — Ein Signal-Handler prüft die Instanzen, die er anfasst, erneut.**
Zwischen dem Auslösen eines Signals und dem Lauf seines Handlers vergeht ein Frame
(`Workspace.SignalBehavior = Deferred`, in neuen Places die Voreinstellung). Was beim
Auslösen noch existierte, kann im Handler zerstört sein.

Der teure Fall ist das Schreiben: `Parent` einer zerstörten Instanz ist **gesperrt**,
eine Zuweisung wirft. *(Korrigiert am 9. August 2026: hier stand, das Lesen einer toten
Instanz liefere `nil`. Das stimmt nur für `.Parent` — andere Properties bleiben lesbar und
liefern ihre letzten Werte. Wer „ist noch da" über einen Property-Read prüft, prüft
nichts.)*

```
The Parent property of X is locked, current parent: NULL
```

Wer im Handler eine Instanz umhängt oder verändert, prüft vorher mit
`instance:IsDescendantOf(game)`, ob sie noch lebt — `instance ~= nil` reicht nicht, die
Referenz bleibt nach `Destroy()` truthy.

> Für zeitkritische Fälle ist ein Signal deshalb nicht immer das richtige Mittel. Eine
> Prüfung pro Frame ist unabhängig vom Zustellzeitpunkt und kostet oft weniger, als es
> klingt.

**R80 — Eigenschaften werden gesetzt, bevor die Instanz geparentet wird.**
Jede Änderung an einer bereits eingehängten Instanz löst Replikation und Change-Events aus.

**R81 — Daten an Instanzen laufen über Attributes, nicht über Value-Objekte.**
Keine Kind-Instanzen im Baum, Replikation inklusive, in Studio direkt editierbar, und
`GetAttributeChangedSignal` statt `.Changed` auf einem Hilfsobjekt.

> Ehrlich dazu: Roblox erklärt `IntValue` und Co. **nicht** für veraltet. Das ist eine
> Hausregel, keine Engine-Vorgabe.
>
> Und die zweite Ehrlichkeit, seit dem 9. August 2026: „Replikation inklusive" ist nur
> dort ein Vorteil, wo der Client den Wert wissen darf. Verborgener Zustand hat als
> Attribut an einer replizierten Instanz nichts verloren — R108.

**R82 — Gruppen von Instanzen werden über `CollectionService`-Tags erkannt, nicht über
Namensabfragen.**
Kein `if part.Name == "Door"`.

**R83 — `WaitForChild` nur mit Timeout, und der `nil`-Fall wird behandelt.**
Ohne Timeout yieldet es unbegrenzt, wenn das Kind nie kommt — der Code hängt dann still,
statt zu scheitern.

#### Aufräummuster ohne Bibliothek

Die üblichen Antworten sind Maid, Janitor oder Trove — externe Module. Stattdessen dieses
Muster, das in jedes Modul passt und keine Abhängigkeit braucht:

```lua
--- Connections held per player, disconnected when they leave
local _connections = {}


--- Tracks a connection so it is cleaned up with the player
local function _track(player: Player, connection: RBXScriptConnection)
	if _connections[player] == nil then
		_connections[player] = {}
	end

	table.insert(_connections[player], connection)
end


--- Disconnects everything held for one player
local function _cleanup(player: Player)
	for _, connection in _connections[player] or {} do
		connection:Disconnect()
	end

	_connections[player] = nil
end
```

Die letzte Zeile ist die wichtigste — sie erledigt genau das, was `Destroy()` nicht tut.

### Teil 4b — Klang und Bild

Asset-Referenzen — Klang-Ids, Bild-Ids — liegen als Daten, nicht als Stringliteral im Code, und
`Id = 0` ist überall der Platzhalter für „noch nicht vorhanden". Klang spielt zusätzlich über **ein**
Modul (den Director); ein Bild wird nur als Property gesetzt.

**R118 — Ein Klang-Eintrag ist eine `{ Id, Volume }`-Table; `Id = 0` heißt „kein Klang".**
Jeder Eintrag in den Klang-Datenmodulen (`Data.SoundIds`, `Data.MusicIds`) trägt die Asset-`Id`
und die `Volume`, mit der er spielt, in **einer** Table — nicht die Id hier und die Lautstärke
dort. `Id = 0` bedeutet, dass noch kein Klang hinterlegt ist: die Abspielstelle bleibt **still,
statt zu crashen**, sodass ein unbefüllter Sound das Spiel nie bricht. Reine Daten, tief
eingefroren (R7/R8); die Schlüssel sind stabile Strings (R9).

**R119 — Klang läuft über einen zentralen Director: `SoundDirector` (Effekte), `MusicController`
(Musik). Kein anderes Modul erzeugt eine `Sound`-Instanz oder ruft `:Play()`.**
Andere Module lösen Klang über die öffentliche API des Directors aus (ein `Play`-Aufruf mit Gruppe
und Schlüssel) — dasselbe Muster wie „nur der `InputController` fasst Eingabe an" (R49) und „nur
der `DataService` spricht mit ProfileStore" (R67). Der Director schlägt den Eintrag nach, verwaltet
die `Sound`-Instanzen (ein wiederverwendeter Stimmen-Pool statt `Instance.new` je Klang) und
**kombiniert die Per-Klang-`Volume` mit der jeweiligen Dämpfung** — etwa einer
`OtherPlayerIntensity`-Dämpfung für fremde Aktionen, der Master-Lautstärke `Config.Music.Volume`
für Musik. Ein **fehlender** Eintrag (Tippfehler im Schlüssel) **wirft**; damit ist gewollte Stille
(`Id = 0`) von einem Fehler unterscheidbar.

**R120 — Eine neue Aktion, die einen Klang tragen könnte, bekommt den Play-Aufruf sofort — auf
einen zunächst leeren Eintrag.**
Wer eine Aktion baut, die logisch ein Geräusch verdient (eine Zelle öffnet, ein Knopf schaltet,
eine Runde endet), verdrahtet den Director-`Play`-Aufruf **gleich mit** und legt dazu einen
Klang-Eintrag mit `Id = 0` an. Der Klang spielt dann standardmäßig **nicht** (R118), aber die
Stelle ist fertig verkabelt: Der Projektleiter füllt später nur die Asset-Id ein, **ohne dass Code
geändert werden muss**. So wächst die Klangkulisse nach und nach, statt dass jede spätere
Vertonung einen eigenen Lauf braucht. *(Prüfung: Ermessen — „könnte einen Klang tragen" ist eine
Auslegung; der Reviewer meldet eine fehlende Vorsorge als HINWEIS, nicht als STANDARD.)*

**R121 — Bild-Ids liegen zentral in `Data.ImageIds`; `Id = 0` heißt „kein Bild"; ein
Platzhalter-Icon ist ein `Id = 0`-Eintrag.**
Jede Bild-Asset-Id (Button-Icons, Brett-Grafiken) liegt in `Data.ImageIds`, im selben `Data`-Ordner
wie `SoundIds`/`MusicIds` — nicht als Stringliteral im UI-Modul. `Id = 0` bedeutet, dass noch kein
Bild hinterlegt ist: der Consumer zeichnet **nichts** statt des Broken-Asset-Markers der Engine. Ein
leeres Platzhalter-Icon ist damit genau ein `Id = 0`-Eintrag — der Projektleiter füllt später nur
die Nummer ein, ohne Code zu ändern (dasselbe Prinzip wie R118/R120 für Klang). Anders als beim
Klang braucht es dafür **kein** zentrales Modul: das UI-Modul liest die Id direkt und setzt sie als
`ImageLabel.Image` (`Id = 0` → `""`), üblicherweise über eine kleine `_imageFor(id)`-Hilfe.

**R122 — Ein Toggle-Button mit Icon trägt zwei Icons: eines für „an", eines für „aus".**
Ein UI-Toggle, der ein Icon zeigt, bekommt in `Data.ImageIds` **zwei** Ids — eine für den aktivierten,
eine für den deaktivierten Zustand (z. B. `{ On = 0, Off = 0 }`) — und wählt beim Zustandswechsel das
passende, an derselben Stelle, an der er auch seine Färbung umschaltet, damit Bild und Farbe synchron
laufen. So liest der Spieler den Zustand auch am Bild ab, nicht nur an der Färbung. Beide Ids starten
als `Id = 0` (Platzhalter nach R121); der Projektleiter füllt sie später. Gilt nur für **Toggles** —
ein Aktions-Knopf (etwa „Runde verlassen") trägt weiter genau ein Icon.

### Teil 5 — Fallstricke beim Testen

Sammelstelle für Verhalten der Werkzeuge, das sich als Fehler im eigenen Code tarnt — oder,
noch gefährlicher, als sauberes Ergebnis. Wer hier stolpert, sucht die Ursache stundenlang
an der falschen Stelle oder hält eine Prüfung für bestanden, die nie gelaufen ist. Deshalb
steht es im Standard und nicht in jemandes Notizen.

**Diese Liste wächst.** Wer beim Arbeiten auf so etwas stößt, trägt es nach.

**R88 — Testcode über `execute_luau` sieht eine frische Modulinstanz, nicht die laufende.**

Ein `require` innerhalb von `execute_luau` liefert **nicht** das Modul, das im laufenden
Spiel arbeitet, sondern eine neu geladene Kopie mit unberührtem Zustand.

*Symptom:* Eine Zustandsabfrage meldet „nicht initialisiert", obwohl das Spiel sichtbar
läuft — etwa `IsGenerated == false`, während `Workspace.Board` mit hunderten Parts
dasteht.

*Falsche Schlussfolgerung:* „Meine Initialisierung ist kaputt."

*Richtiges Vorgehen:* Die beiden Prüfarten trennen.
- **Unit-Test auf der frischen Instanz** — zulässig, solange das Modul dabei nichts an der
  Welt verändert. Gut für reine Rechenlogik, Zustandsübergänge, Tabellenverwaltung.
- **Prüfung am laufenden Spiel** — über den Instanzbaum, über die echten Remotes, über
  `get_console_output`. Alles, was den tatsächlichen Zustand betrifft, läuft so.

Wer beides vermischt, misst zwei verschiedene Programme.

**Und im Edit-DataModel ist diese Kopie zusätzlich veraltet.** Der `require`-Cache überlebt
dort **Bearbeitungen** und **getrennte `execute_luau`-Aufrufe**: Nach einem `multi_edit`
liefert ein `require` weiterhin die Fassung, die vor der Änderung geladen wurde — über
Minuten und über beliebig viele Werkzeugaufrufe hinweg.

*Symptom:* `script_read` zeigt `Width = 25`, ein `require(Config)` meldet unbeirrt `20`.

*Warum es teuer ist:* Das Ergebnis ist **plausibel**. Es ist kein Fehler, keine Warnung, kein
`nil` — es ist eine gültige frühere Fassung. Zwei Agenten dieses Projekts sind darauf
hereingefallen; der zweite hat daraus eine Fundmeldung gebaut, die Quelle sei beschädigt, und
hat einen halben Bericht darauf gestützt.

> *Richtiges Vorgehen:* **Ein Konfigurationswert, der eine Messung trägt, wird über
> `script_read` aus der Quelle belegt, nie über `require`.**

Braucht der Testcode das Modul als Objekt, hilft ein Umweg: das ModuleScript klonen, den
**Klon** requiren, den Klon zerstören. Der Klon ist neu und hat keinen Cache-Eintrag.

**R90 — Eine Zustandsmessung gehört in dasselbe `execute_luau` wie die auslösende
Aktion.**

Zwischen zwei MCP-Aufrufen vergeht **echte Wanduhrzeit** — Sekunden, nicht Millisekunden.
In dieser Zeit läuft das Spiel weiter: Figuren respawnen, Zeitgeber laufen ab,
Drosselfenster öffnen sich wieder.

*Symptom:* Nach einem Klick auf eine Minenzelle meldet die Prüfung `Health = 100`.

*Falsche Schlussfolgerung:* „Der Todesfall wird verschluckt."

*Tatsächlich:* Die Figur ist längst respawnt — der Standardwert dafür liegt bei fünf
Sekunden, und genau so lange kann ein separater Werkzeugaufruf dauern.

*Richtiges Vorgehen:* Auslöser und Messung in **einem** Skript, mit der Wartezeit
dazwischen im Skript selbst. Was zwei Aufrufe braucht, misst zwei verschiedene Zeitpunkte.

**R93 — `script_grep` taugt nicht als alleiniger Nachweis.**

Vier bestätigte Eigenheiten, alle in dieselbe Richtung gefährlich: sie lassen eine Prüfung
bestanden aussehen, die nichts geprüft hat.

| Eigenheit | Folge |
| :--- | :--- |
| **Case-insensitiv** | `^local %l` findet auch `local Players`. Jede Prüfung auf Groß- oder Kleinschreibung ist damit **nicht entscheidbar**. |
| **`%f`-Frontier schlägt fehl** | `%f[%w]Config%f[%W]` liefert **null** Treffer, obwohl `Config` dutzendfach vorkommt. Das Muster wird nicht ignoriert, es scheitert stumm — und null Treffer sehen aus wie ein sauberes Ergebnis. |
| **Zeilennummern zählen keine Leerzeilen** | Der Versatz gegen `script_read` wächst mit jeder Leerzeile davor. Bei einem Standard, der zwei Leerzeilen zwischen Funktionen verlangt, ist er **systematisch groß**. |
| **Runde Klammern sind Captures** | `warn(` bricht mit `unfinished capture` ab — laut und harmlos. **Balancierte** Klammern dagegen werden stumm als Capture übersetzt: `Vector3.new(0, 0, 0)` liefert null Treffer, obwohl die Zeile wortwörtlich existiert. Klammern immer als `%(` und `%)` maskieren. |
| **Abbruch nach 50 Treffern** | Wer nur die Trefferzahl liest, hält eine abgeschnittene Suche für vollständig. |

*Richtiges Vorgehen:* Eine Suche, deren Ergebnis eine Abnahme trägt, wird **auf einen Pfad
eingeschränkt** und ihr Muster wird an einem bekannten Positivfall gegengeprüft. Wo
Groß-/Kleinschreibung zählt, hilft nur Lesen. Zeilennummern aus `script_grep` niemals
ungeprüft in einen Bericht übernehmen — sie stimmen nicht.

**R95 — Zeitmessungen über die Werkzeugkette tragen einen Aufschlag.**

Eine Eingabe, die über die MCP-Werkzeuge synthetisiert wird, kommt nicht mit der
angeforderten Dauer im Spiel an. Gemessen wurde ein **konstanter Aufschlag von rund 0,28
Sekunden**: 20 ms Anweisung erschienen als 0,299 s, 90 ms als 0,399 s — die Differenz
zwischen beiden stimmte auf 70 ms.

*Folge:* Wer eine Zeitschwelle im Emulator prüft, misst systematisch zu lang. Eine Schwelle
von 0,4 s ist dort schon grenzwertig, obwohl sie auf einem Gerät bequem wäre.

*Richtiges Vorgehen:* Den Aufschlag einmal bestimmen — zwei Anweisungen mit bekanntem
Abstand, die Differenz der gemessenen Werte gegen die Differenz der angeforderten —, und
ihn danach herausrechnen. Was übrig bleibt, ist Spielerverhalten. Was man ungerechnet
übernimmt, ist Latenz der Testautomation.

**R104 — Der Bezugspunkt einer Messung darf nicht aus der Messmenge stammen.**

Wer alle Werte gegen den kleinsten der eigenen Reihe normiert, bekommt garantiert eine Null
— und die beweist nur, dass er der kleinste **dieser Reihe** ist, nie dass außerhalb nichts
kleiner war. **Eine Auswertung, die kein Gegenbeispiel finden *kann*, ist keine Prüfung.**

*Der Fall dahinter.* Eine Flutanimation sollte von der angeklickten Kachel nach außen
laufen. Die Messreihe war sorgfältig — echtes Remote, laufender Client, Stapel absichtlich
in umgekehrter Reihenfolge übergeben —, die Zahlen stimmten, die Monotonie über neun
Abstandsringe war echt. In der Tabelle stand `Abstand 0 → 0 ms`, und daraus wurde
geschlossen, die angeklickte Kachel laufe zuerst. Tatsächlich war sie **gar nicht in der
Messmenge**: die Null war der normierte Minimalwert der übrigen Ringe. Die geklickte Kachel
sank in Wahrheit **zuletzt**, weil ihre Druck-Rückmeldung sie 170 ms lang festhielt. Gefunden
hat es niemand am Bild, sondern eine Nachrechnung im Review.

*Richtiges Vorgehen:* Bezugspunkt ist ein Ereignis, das **unabhängig von der Messmenge**
feststeht — der Moment der Eingabe, ein Zeitstempel vor dem Auslöser, ein Wert aus der
Konfiguration. Und das Urteil wird gegen den **Extremwert** der Vergleichsgruppe gefällt,
nicht gegen ihren Mittelwert: Ein Mittelwert verdeckt die eine Ausreißerzelle, die die
Aussage widerlegt.

*Die allgemeine Form:* Vor jeder Messreihe steht die Frage, **welches Ergebnis die Methode
nicht liefern könnte**. Wer sie nicht beantworten kann, misst noch nicht.

**R96 — Eine hängende Berührung im Emulator legt die Kamera still, ohne es zu melden.**

Nach einem Rechtsklick-Zyklus im Device-Emulator bekommt die linke Berührung **kein
`End`**. Der zurückbleibende Touch-Punkt schaltet Roblox' Kamerasteuerung ab — lautlos.

*Symptom:* Eine Messreihe über Züge von 30, 45 und 150 Pixeln ergibt **exakt 0,0000°
Kameradrehung**. Das sieht aus wie ein sauberes Ergebnis und ist eines der gefährlichsten
Messartefakte dieser Umgebung: nicht „Fehler", sondern „alles null".

*Richtiges Vorgehen:* Vor jeder Messung, die auf Kamera oder Eingabezustand beruht, den
**Play-Modus neu starten**. Und eine Messreihe, die durchgehend exakt null liefert, ist
zuerst als Artefakt zu verdächtigen und nicht als Befund zu berichten.

**R97 — Synthetische Zeigereingaben kommen versetzt an. Vor der ersten Kompensation messen.**

Eine über `user_mouse_input` angeforderte Position erreicht das Spiel nicht dort, wo sie
angefordert wurde. Die Ursprungsmessung über elf Punkte ergab **konstanten Versatz in X,
exakt null in Y**:

> **Ankunft = Anforderung − 62**
>
> Wer auf einen Punkt zielen will, **addiert** den Kalibrierwert auf den gewünschten
> Ankunftspunkt.

Das Vorzeichen ist der teure Teil dieser Regel. Wer es dreht, liegt 124 Pixel daneben und
sucht den Fehler in seinem Handler.

> ### ⚠ Nachtrag vom 6. August 2026 — die −62 ist **kein Erwartungswert**
>
> Mehrere spätere Läufe in derselben Studio-Installation haben etwas anderes gemessen, und
> zwar nicht nur eine andere Zahl, sondern eine **andere Achse**:
>
> | Sitzung | Δx | Δy |
> | :--- | ---: | ---: |
> | Ursprungsmessung dieser Regel | **−62** | 0 |
> | mehrere Sitzungen im August 2026 | **0** | **+58** |
>
> **Wer die −62 als Erwartungswert liest und damit kompensiert, verfehlt.** Genau das ist
> dreimal passiert, jedes Mal einem Agenten, der die Regel korrekt befolgt hat: Er addierte
> 62 auf X, und der Klick ging daneben — in einer Sitzung, in der X gar nicht versetzt war.
> Der Satz weiter unten, die 62 seien „vermutlich an dieses Studio-Fenster gebunden", stand
> schon vorher da. Er hat nicht gereicht, weil die Zahl im Blockzitat steht und der Vorbehalt
> im Fließtext.
>
> *Richtiges Vorgehen:* **Vor der ersten Kompensation messen, in beiden Achsen.** Das
> Ergebnis kann null sein; dann wird nicht kompensiert. Das Blockzitat oben ist die Messung
> **einer** Sitzung und zeigt die Form des Ergebnisses, nicht den Wert zum Einsetzen.
>
> *Und die Gegenprobe wird gegenstandslos, wenn der Versatz null ist.* Der Absatz weiter
> unten verlangt „einen absichtlichen Fehlschlag als Positivkontrolle: eine Anforderung ohne
> Kompensation, die daneben gehen *muss*". Bei Δ = 0 **kann** sie nicht danebengehen — und
> nach R104 ist eine Auswertung, die kein Gegenbeispiel finden kann, keine Prüfung. Ein
> gemessenes Δ = 0 wird deshalb als **Kalibrierergebnis** notiert und nicht als bestandene
> Kontrolle. Wer in dieser Lage trotzdem eine Aussage über die Abbildung braucht, muss sie
> anders bauen — etwa über eine Fläche, die kleiner ist als der behauptete Versatz.
>
> *Eine offene Frage, die dieser Nachtrag nicht beantwortet:* In den Sitzungen mit Δy = +58
> entsprach der Versatz **genau dem GUI-Inset**. Ob der Eingabeweg dort einen Inset
> verrechnet — was R98 für die Ursprungsmessung ausdrücklich ausschließt — oder ob die
> angeforderte Koordinate in diesen Sitzungen in einem anderen Bezugsrahmen gelesen wurde,
> ist **nicht geklärt**. Beobachtung, keine Feststellung. Für die Praxis ändert es nichts:
> erst messen, dann kompensieren.
>
> **Nachtrag vom 6. August 2026 — kalibriert wird gegen die Art von Ziel, die man treffen
> will.** In einer Sitzung traf ein **Weltziel** erst mit `Anforderung = Ziel − 58` in Y, ein
> **GUI-Knopf** derselben Sitzung dagegen mit der **unkorrigierten** `AbsolutePosition`-Mitte
> — mit der 58er-Korrektur ging er daneben, ohne saß er.
>
> Das ist unmittelbar praktisch: Das Verfahren weiter unten schickt eine Anforderung auf die
> Mitte einer **GUI**-Fläche und liest die Ankunft aus `InputBegan`. Nach dieser Beobachtung
> liefert genau diese Sonde **null** — und wer die Null danach auf einen **Weltklick**
> anwendet, geht um den Inset daneben und sucht den Fehler in seinem eigenen Handler. Zwei
> Läufe haben daran Zeit verloren.
>
> Eine Differenz zwischen beiden ist **kein zweiter Versatz, den man pflegt**, sondern ein
> Hinweis darauf, **wo** der Versatz sitzt: Ist der Eingabeweg für beide derselbe und braucht
> nur einer eine Korrektur, liegt sie nicht im Eingabeweg, sondern in der **lesenden** API,
> über die gezielt wird — beim Weltziel in aller Regel `GetMouseLocation()`, siehe R98.
>
> *Beobachtung einer Sitzung, ohne Positivkontrolle.* Sie beantwortet die offene Frage des
> vorigen Absatzes nicht, sie schränkt sie ein.

*Herkunft, als Indiz und nicht als Rechenweg:* Der unberührte Zeiger meldet `(-63, -1)`,
der gemessene Versatz ist `(-62, 0)`. Beide Achsen um genau eins daneben. Das Modell
„Fensterursprung liegt links außerhalb des Viewports" erklärt die **Größenordnung**, aber
nicht den **Wert** — und genau deshalb wird kalibriert statt gerechnet. Die 62 sind
vermutlich an dieses Studio-Fenster gebunden und in einer anderen Sitzung eine andere Zahl.

*Richtiges Vorgehen:* Einmal pro Sitzung eine Anforderung auf die bekannte **Mitte einer
Fläche mit bekannter `AbsolutePosition`** schicken und die Ankunft aus `TouchStarted` bzw.
`InputBegan` dagegenhalten. Die Differenz ist der Kalibrierwert für den Rest der Sitzung.

**Und der naheliegende Ausweg funktioniert nicht: Ein `instance_path` verrechnet den Versatz
nicht.** Er wird auch dann auf die angeforderte Bildschirmposition angewandt. Gemessen: ein
Klick auf die Mitte eines 78 px breiten Knopfs bei x = 172 kam bei x = 110 an, also 23 px
neben dem Knopf, und löste nichts aus; mit `172 + 62` traf er. Wer die Instanz angibt, hält
das für die sichere Variante — sie ist es nicht, sie ist nur die, bei der man den Fehlschlag
für ein Problem des Knopfs hält.

*Und die Gegenprobe gehört dazu:* Eine Messreihe, in der jede Anforderung trifft, belegt
nicht, dass die Abbildung stimmt — sie belegt, dass die Flächen groß sind. Wer einen
Versatz nachweisen will, baut **einen absichtlichen Fehlschlag als Positivkontrolle**: eine
Anforderung ohne Kompensation, die daneben gehen *muss*. Trifft sie doch, war die Annahme
falsch. Eine Kontrolle taugt umso mehr, je knapper sie verfehlt — knapper als die
Sicherheitsmarge, die der Versuch belegen soll.

**R98 — Der GUI-Inset ist eine Differenz zwischen lesenden APIs, nicht Teil des Eingabewegs.**

Die beiden Fallstricke sehen verwandt aus und sind es nicht. Der Versatz aus R97 liegt in
**X**, ist konstant und additiv und betrifft den **Eingabeweg**. Der GUI-Inset liegt in
**Y** und ist eine Differenz zwischen APIs, die eine Position **lesen**.

*Nachgewiesen durch Gegenprobe:* Eine Anforderung, bei der der Inset absichtlich in Y
abgezogen wurde, landete um genau den Inset zu hoch und traf nicht. Enthielte die
Eingabeabbildung eine Inset-Korrektur, wäre dieser Versuch ein Treffer gewesen. Zusammen
mit Δy = 0 über elf Punkte gilt: **auf dem Eingabeweg wird kein Inset verrechnet.**

*Und daher rührt die Falle, die der R97-Nachtrag vom 6. August 2026 beschreibt:* Weil der
Inset im **Leseweg** sitzt, hängt der nötige Korrekturwert daran, **worauf** gezielt wird —
ein Weltziel über `GetMouseLocation()` braucht ihn, ein GUI-Ziel über `AbsolutePosition`
nicht. Eine Kalibrierung an der einen Zielart gilt nicht für die andere.

*Richtiges Vorgehen:* Wo zwischen zwei Koordinatenräumen umgerechnet werden muss, immer
`GuiService:GetGuiInset()` abfragen — **nie eine gemessene Zahl festschreiben.** Der Wert
hängt von Plattform und Oberflächenzustand ab; die im Emulator gemessenen 58 Pixel sind
eine Beobachtung dieser Umgebung, kein Wert für den Code.

**Und er wird in dem Moment gelesen, in dem er gebraucht wird — nicht in `Init` oder
`Start` und dann behalten.** Der Wert hängt am Zustand der Oberfläche, und der steht während
des Hochlaufs noch nicht fest. Einmal gelesen und aufgehoben ist eine Abfrage dasselbe wie
eine festgeschriebene Zahl, nur schwerer zu erkennen: die Stelle sieht abgesichert aus.

*(Beobachtet, nicht abschließend belegt: in der `Start`-Phase des Clients antwortete
dieselbe Abfrage 0, im laufenden Spiel 58. Die Regel trägt auch ohne diesen Befund — ein
zur Benutzungszeit geholter Wert ist unabhängig davon richtig.)*

Ebenso: **eine Fläche, die den ganzen Bildschirm abdecken soll, kann das in einer
`ScreenGui` mit `IgnoreGuiInset = false` nicht.** `UDim2.fromScale(1, 1)` misst dort den um
den Inset verkleinerten Raum, und die Differenz bleibt als unbedeckter Streifen stehen.
Wer eine modale Rückwand baut, setzt `IgnoreGuiInset = true` — **eine bildschirmfüllende
Fläche *ist* ein Umrechnungsfall**, auch wenn im Modul keine zweite Koordinate vorkommt.

**R99 — Eine Plattformweiche hängt nie allein an `MouseEnabled`, `KeyboardEnabled` oder
`TouchEnabled`.**

Der Device-Emulator meldet **alle drei gleichzeitig als `true`**, während er ausschließlich
Eingaben vom Typ `Touch` erzeugt.

*Folge:* Eine Weiche auf diesen Merkmalen steht im Emulator **immer auf „Maus"** — und zwar
unabhängig davon, ob sie richtig oder falsch geschrieben ist. Sie ist dort weder prüfbar
noch widerlegbar. Ein Fehler an dieser Stelle überlebt jeden Emulator-Test und fällt erst
auf einem echten Gerät auf.

Die Merkmale beantworten auch die falsche Frage. Sie sagen, was ein Gerät **kann**, nicht,
womit gerade gespielt wird. Ein Notebook mit Touchscreen kann beides, und die Antwort
wechselt im Betrieb — ein Gamepad wird eingesteckt, ein Convertible wird zugeklappt.

*Richtiges Vorgehen:* Wer unterscheiden muss, welches Gerät **gerade führt**, merkt sich das
aus den tatsächlich eingehenden Eingaben — welcher Eingabetyp zuletzt gehandelt hat —
statt eine Fähigkeit abzufragen. Das ist zugleich die einzige Form, die im Emulator prüfbar
ist, weil sie sich an denselben Ereignissen entscheidet, die der Emulator erzeugt.

Die Merkmale bleiben brauchbar für die Frage, ob eine Bedienform **überhaupt** angeboten
wird. Sie taugen nicht für die Frage, welche gerade benutzt wird.

**R100 — Im Play-Modus sehen die lesenden Werkzeuge nur einen der beiden DataModels.**

Welcher es ist, **steht nicht im Ergebnis** und kann zwischen zwei Aufrufen wechseln.

Betroffen sind **alle** lesenden Werkzeuge, nicht nur `script_grep`: `script_read` scheitert
mit *Script not found* an einem Skript, das im anderen Zweig liegt; `script_search` findet
bekannte Skripte nicht; `search_game_tree` meldet einen Dienst als kinderlos, der voll ist.
Keine dieser Meldungen sagt „falscher DataModel" — sie sagen „gibt es nicht".

*Symptom:* Eine Suche nach `MaxRevealDistance` lieferte drei Treffer — aus `Config` und
zweimal aus dem Client. Der Treffer im serverseitigen `ValidationService` fiel **stumm**
aus, obwohl die Zeile dort nachweislich steht. Die naheliegende Schlussfolgerung wäre
gewesen: „der Server prüft die Entfernung nicht." Ein `script_search` auf dasselbe
Serverskript lieferte im selben Zustand **null** Treffer bei existierendem Skript.

Das ist derselbe Fehlertyp wie R93 und einen Grad tückischer: Dort scheitert ein Muster,
hier scheitert die **Auswahl des Suchraums** — und zwar unbemerkt und nicht reproduzierbar.
Wenige Aufrufe später lieferte dieselbe Sitzung Treffer aus dem Serverzweig und dafür keine
aus dem Client.

*Richtiges Vorgehen:* Suchen, deren Ergebnis eine Abnahme trägt, laufen im **Edit-Modus**.
Muss im Play-Modus gesucht werden, wird das Ergebnis über einen bekannten Positivfall **in
genau dem Zweig** gegengeprüft, um den es geht — ein Positivfall aus dem falschen DataModel
beweist nichts.

**R102 — Bei `multi_edit` sucht jede Ersetzung im *Ergebnis* der vorherigen, nicht im
Ausgangstext.**

Wer in einem Aufruf erst einen Funktionsrumpf anlegt und danach denselben Text an seiner
ursprünglichen Stelle ersetzen will, trifft **seinen eigenen frischen Rumpf** — die neue
Funktion ruft sich selbst auf, die eigentliche Zielstelle bleibt unverändert.

*Der reale Fall:* Zwei `SetAttribute`-Zeilen sollten aus einer Funktion in eine neu
angelegte Hilfsfunktion wandern. Die Ersetzung traf das erste Vorkommen — und das lag seit
der vorangegangenen Ersetzung im Rumpf der Hilfsfunktion selbst. Ergebnis: eine
Endlosrekursion und eine Aufrufstelle, die nie umgestellt wurde.

Der Fehler ist besonders tückisch, weil er **beide** Hälften des Vorgangs falsch macht und
trotzdem plausibel aussieht: Die neue Funktion existiert, sie enthält den erwarteten Code,
und die alte Stelle sieht aus wie vor der Bearbeitung.

*Richtiges Vorgehen:* Ersetzungen, die denselben Text mehrfach betreffen könnten, **eindeutig
verankern** — genug Kontextzeilen mitnehmen, dass nur die gemeinte Stelle passt. Und nach
einem Aufruf, der Code verschiebt statt nur ändert, **die Datei zurücklesen**. Nicht greppen:
lesen. Ein Grep auf den verschobenen Text findet ihn in beiden Fällen.

*Und wie weit gelesen wird:* **Nach einer Ersetzung, die nichts verschoben hat, reicht die
Stelle plus etwa zwanzig Zeilen Umgebung** — es ist eine Kontrollfrage, nicht eine
Verständnisfrage. `script_read` nimmt dafür `start_line_one_indexed` und
`end_line_one_indexed_inclusive`; sein Standard liest die **ganze** Datei, und bei tausend
Zeilen kostet das rund 15 000 Tokens, um drei Zeilen zu prüfen.

**Vollständig gelesen wird weiterhin**, wo es ums Verstehen geht: eine Datei, die man zum
ersten Mal anfasst, eine Architekturfrage, ein Edit, der Code verschoben hat. Der Unterschied
ist die Absicht — Kontrolle liest gezielt, Verständnis liest ganz. Wer hier zu scharf spart,
spart die Funde weg: Mehr als ein Fund dieses Projekts stammt daher, dass jemand beim
Zurücklesen etwas sah, wonach er nicht gesucht hatte.

**R105 — Wer einen Punkt der Restliste erledigt, hakt ihn ab. Wer aus ihr beauftragt, prüft
ihn vorher.**

Jedes dieser Projekte führt eine Restliste — ein Dokument, in dem Läufe und Reviews sammeln,
was offen blieb. Sie ist die Grundlage, aus der später Aufträge geschnitten werden, und
deshalb ist ein **falsch gebliebener Eintrag teurer als gar keiner**: Er beauftragt Arbeit,
die es nicht mehr gibt, und verbraucht das Vertrauen in alle Einträge daneben.

*Zwei Pflichten, an zwei verschiedenen Stellen:*

**Wer einen Punkt erledigt — auch nebenbei, auch ungefragt —, sagt es im Bericht.** Ein Lauf,
der einen fremden Kommentar mitzieht oder einen toten Wert anschließt, hat damit einen
Listenpunkt geschlossen. Eintragen tut es der Lead; wissen kann er es nur, wenn es im Bericht
steht. Das ist dieselbe Bringschuld wie bei R101, nur zum Dokument statt zum Nachbarmodul.

**Wer aus der Liste beauftragt, prüft jeden Punkt am heutigen Code, bevor er ihn in einen
Auftrag schreibt.** Ein Eintrag ist ein Befund von damals, keine Zusage über heute.

*Gemessen, 6. August 2026:* Ein Sammellauf über neun Kommentargruppen und sechs angeblich
tote Werte fand **die Hälfte der Liste überholt** — von fünf Kommentaren mit einer widerlegten
Annahme stand einer noch falsch da, drei der sechs Werte hatten längst wieder einen Leser,
einer davon seit demselben Abend. Keiner der Läufe, die das erledigt hatten, hatte es gemeldet.
Das Nachprüfen kostete einen ganzen Lauf.

*Und die Gegenrichtung gehört dazu:* Ein Eintrag, der sich als **weiterhin gültig** erweist,
ist ebenfalls ein Ergebnis. Wer ihn prüft und stehen lässt, vermerkt das Datum der Prüfung —
sonst prüft ihn der nächste Lauf erneut.

*Zwei Mittel, die das Nachprüfen billig machen:*

**Ein Eintrag, dessen Wahrheit von einer Suche abhängt, trägt seinen Prüfbefehl mit.** Statt
„`FxCullDistance` ist ungelesen" schreibt die Liste: *„`FxCullDistance` ungelesen — Beleg:
`script_grep FxCullDistance`, 2 Treffer, beide Definition und Kommentar, 6.8.2026."* Der
nächste Lauf muss den Punkt dann nicht **nachprüfen**, sondern nur den Befehl **noch einmal
ausführen** — Sekunden statt einer halben Sitzung, und die Abweichung fällt am Anfang auf statt
am Ende. Kosten: ein halber Satz je Eintrag.

**Der Coder-Bericht führt „Was ich in der Restliste widerlegt habe" als eigenes Feld.** Es gibt
Felder für bewusste Abweichungen und für die Vertrauensgrenze, aber keines dafür, dass ein Lauf
ein Dokument überholt hat — und genau das passiert beiläufig. Leer bleiben darf es; ungenannt
bleiben darf ein widerlegter Eintrag nicht.

**R106 — Wer eine Messzahl in einem Kommentar neu misst, misst jede Zahl desselben Absatzes
neu — oder schreibt an die stehen gebliebene ihre Messbedingung.**

Ein Absatz, der zur Hälfte neu gemessen ist, sieht ganz neu gemessen aus. Die stehen gebliebene
Zahl **erbt die Glaubwürdigkeit der neuen daneben**, ohne sie zu verdienen — und ein Diff zeigt
sie nicht, weil sich an ihr nichts geändert hat. Der nächste Leser hat keinen Anlass zu prüfen,
was aussieht, als sei es gerade geprüft worden.

*Gemessen, 6. August 2026:* Ein Sammellauf vermaß den Kopf von `Config.BoardNoise` neu, weil
zwei Regler sich geändert hatten. Ein Satz des Absatzes blieb stehen — er gehörte zur alten
Konfiguration und beschreibt seither ein Medianbrett, das den Fall gar nicht mehr hat. Gefunden
hat ihn der Reviewer, nicht der Diff.

*Und die Ausnahme, die die Regel praktikabel macht:* Eine Zahl, die von den geänderten Reglern
**nachweislich unabhängig** ist, darf stehen bleiben — dann gehört der Nachweis dazu, nicht das
Schweigen. Im selben Absatz traf das auf „die mittlere Zelle steht auf halber Höhe" zu, weil die
Rauschfunktion symmetrisch ist.

### Teil 6 — Nachweispflichten

Zwei Lücken, die kein Codefund je zeigt, weil sie im **Vorgehen** liegen — beide am
9. August 2026 aus einer externen Prüfung übernommen. Der gemeinsame Nenner: Coder und
Reviewer lesen denselben Standard, und was der eine übersieht, übersieht der andere aus
demselben Grund. Gegen diese geteilte Blindheit helfen keine weiteren Regeln über Code,
sondern nur unabhängige Erhebung und benannte Evidenz.

**R114 — Der Reviewer erhebt das Remote-Inventar unabhängig aus dem Code.**
Wer die Vertrauensgrenze prüft, zählt selbst: jedes `FireServer`, jedes `OnServerEvent`,
jede `RemoteFunction` — aus dem Code erhoben (unter den Vorbehalten von R93 und R100),
nicht aus dem Coder-Bericht übernommen. Die eigene Liste wird gegen die des Berichts
gehalten; jede Differenz ist ein Fund, in beide Richtungen. Ein Remote, das der Bericht
nicht erwähnt, ist der teuerste Eintrag der ganzen Prüfung — und genau der, den ein
berichtsgläubiger Review strukturell nie sieht.

**R115 — Jede Änderung nennt im Bericht die Evidenz, mit der sie geprüft wurde — und die
Evidenzklasse passt zur Änderungsklasse.**
Reine Rechenlogik: Unit-Test auf der frischen Modulinstanz (R88). Input-, Kamera- und
UI-Pfade: ein dynamischer Lauf im Emulator oder auf dem Gerät — drei statische Reviews
haben den R94-Bug für korrekt erklärt, gefunden hat ihn der erste Gerätelauf. Ökonomie-
und Datenpfade: der Doppelfeuer-Fall (R109) und der Verlassen-Fall (R76), ausdrücklich.
„Statisch gelesen, sah richtig aus" ist für diese Klassen keine Evidenz, sondern das
Protokoll ihres bekanntesten Versagens.

---

## Referenzbeispiel

```lua
--- Handles loading, caching and saving of player profiles.


---------------
--- IMPORTS ---
---------------

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")


local Types = require(ReplicatedStorage.Shared.Types)


-------------
--- TYPES ---
-------------

--- Tracks how far a profile has progressed through loading
type LoadState = "pending" | "ready" | "failed"


-------------
--- LOCAL ---
-------------

--- Constants
local DEFAULT_COINS = 0
local DEFAULT_LEVEL = 1


--- Cached profiles and their load state, keyed by player
local _profiles = {}
local _loadStates = {}


--- Creates a fresh profile for a player without saved data
local function _buildDefaultProfile(): Types.Profile
	return {
		Coins = DEFAULT_COINS,
		Level = DEFAULT_LEVEL,
	}
end


--- Warms the cache for a joining player
local function _loadProfile(player: Player)
	_profiles[player] = _buildDefaultProfile()
	_loadStates[player] = "ready"
end


--- Drops cached data when a player leaves
local function _clearProfile(player: Player)
	_profiles[player] = nil
	_loadStates[player] = nil
end


--------------
--- PUBLIC ---
--------------

local DataService = {}


--- Reports how far the player's profile has progressed through loading
function DataService.GetLoadState(player: Player): LoadState
	return _loadStates[player] or "pending"
end


--- Returns the player's profile, creating a default one if none exists
function DataService.GetProfile(player: Player): Types.Profile
	if not _profiles[player] then
		_profiles[player] = _buildDefaultProfile()
		_loadStates[player] = "ready"
	end

	return _profiles[player]
end


--- Wires up player lifecycle handling, including players already present (R110)
function DataService.Start()
	Players.PlayerAdded:Connect(_loadProfile)
	Players.PlayerRemoving:Connect(_clearProfile)

	for _, player in Players:GetPlayers() do
		_loadProfile(player)
	end
end


return DataService
```

---

## Regelindex

Nachschlagetabelle für zitierte Regelnummern. Die Spalte **Prüfung** steuert die
Einstufung im Review (siehe „Abweichungen"): *mechanisch* → `STANDARD`,
*Ermessen* → `HINWEIS`, *Verfahren* → `STANDARD`, zitiert gegen Bericht oder Messaufbau
statt gegen eine Codezeile.

| Nr. | Regel | Prüfung |
| :--- | :--- | :--- |
| R1 | Genau ein Einstiegspunkt pro Seite | mechanisch |
| R2 | Zweiphasiger Start: `Init` baut auf, `Start` ruft auf | mechanisch |
| R3 | `Shared` ist zustandslos | mechanisch |
| R4 | Keine require-Ketten quer durch den Baum | mechanisch |
| R5 | Remotes an genau einer Stelle definiert | mechanisch |
| R6 | Ein Modul, eine Zuständigkeit | Ermessen |
| R7 | Config und Data: reine Daten, keine Logik | mechanisch |
| R8 | Zurückgegebene Tabellen tief versiegeln (`Util.deepFreeze` oder manuell je Ebene) — `table.freeze` allein friert flach | mechanisch |
| R9 | Data-Einträge, auf die verwiesen wird, mit stabilen String-IDs als Schlüssel | mechanisch |
| R10 | Nichts Geheimes unter `Shared` — Serverseitiges nach `ServerUtils` | Ermessen |
| R11 | IMPORTS enthält nur Requires und `game:GetService()` | mechanisch |
| R12 | LOCAL enthält modulinternen Code | mechanisch |
| R13 | PUBLIC enthält alles von außen Zugreifbare | mechanisch |
| R14 | BOOT nur in Einstiegspunkten | mechanisch |
| R15 | Nicht jede Modulsorte hat alle Blöcke | mechanisch |
| R16 | Der gesamte Code ist englisch, einschließlich Instanznamen im DataModel | mechanisch |
| R17 | Der Unterstrich markiert den Scope, nicht die Sichtbarkeit | mechanisch |
| R18 | Konstanten `SCREAMING_SNAKE_CASE` unter `--- Constants` | mechanisch |
| R19 | Namen geben ein klares Bild, Ein-Zeichen-Namen nur im Funktions-Scope | Ermessen |
| R20 | Module PascalCase, Substantiv, kein Präfix | mechanisch |
| R21 | Kommentare sind englisch und knapp | Ermessen |
| R22 | Funktionskommentar mit `---` direkt über der Funktion — **Pflicht nur, wo es etwas zu begründen gibt**; sagt der Name alles, bleibt er weg | Ermessen |
| R23 | `---`-Kommentar über jedem Variablenblock, ein Satz | mechanisch |
| R24 | Keine Kommentare innerhalb von Funktionen | mechanisch |
| R25 | Kommentare erklären *warum*, nicht *was* — und wo es kein Warum gibt, keinen Kommentar | Ermessen |
| R26 | Zwei Leerzeilen zwischen Funktionen | mechanisch |
| R27 | Zwei Leerzeilen nach einem Variablenblock | mechanisch |
| R28 | Öffentliche Funktionen sind annotiert (Parameter und Rückgabe) | mechanisch |
| R29 | `--!strict` freiwillig, erwünscht in Hilfs- und Datenmodulen | mechanisch |
| R30 | Geteilte Typen wandern nach `Shared.Types` und werden `export` | mechanisch |
| R31 | Nur der Einstiegspunkt requiret Logik-Module | mechanisch |
| R32 | Hilfs- und Datenmodule werden direkt requiret | mechanisch |
| R33 | Abhängigkeiten über `Init(modules)`, im LOCAL-Block abgelegt | mechanisch |
| R34 | Innerhalb einer Phase ist die Reihenfolge undefiniert | Ermessen |
| R35 | `Init` und `Start` sind optional | mechanisch |
| R36 | Der Loader fängt keine Fehler ab | mechanisch |
| R37 | Der Modulordner ist flach | mechanisch |
| R38 | TYPES-Block direkt nach IMPORTS, nur wenn Typen definiert werden | mechanisch |
| R39 | Typnamen immer PascalCase, ohne Unterstrich | mechanisch |
| R40 | LOCAL-Reihenfolge: Konstanten → injizierte Module → Zustand → Funktionen | mechanisch |
| R41 | Block-Kommentar: zwei Leerzeilen darüber, eine darunter | mechanisch |
| R42 | Höchstens eine Leerzeile innerhalb eines Funktionsrumpfs | mechanisch |
| R43 | Einrückung mit Tabs | mechanisch |
| R44 | Maximale Zeilenlänge 120 Zeichen | mechanisch |
| R45 | Doppelte Anführungszeichen | mechanisch |
| R46 | Abschließendes Komma in mehrzeiligen Tabellen | mechanisch |
| R47 | Keine Semikolons, eine Anweisung pro Zeile | mechanisch |
| R48 | Kein globales Symbol, alles `local` | mechanisch |
| R49 | Nur der Input-Controller berührt eine Input-API | mechanisch |
| R50 | CAS für Aktionen, UIS nur für Abfragen, `GetMouse()` verboten | mechanisch |
| R51 | Aktionsnamen sind Konstanten, keine Stringliterale | mechanisch |
| R52 | Jede Aktion deckt Tastatur/Maus, Gamepad und Touch ab | mechanisch |
| R53 | Handler geben `Sink` oder `Pass` explizit zurück | mechanisch |
| R54 | Moduswechsel über Binden/Entbinden, nicht über Flags | mechanisch |
| R55 | Zu jeder Bindung gehört eine Entbindung | mechanisch |
| R56 | Signale als zur Laufzeit erzeugtes `BindableEvent`, nur `.Event` öffentlich | mechanisch |
| R57 | Keine Tabellen über Signale schicken | mechanisch |
| R58 | Der Client meldet Absichten, nie Ergebnisse | mechanisch |
| R59 | Jedes Remote-Argument wird serverseitig validiert (inkl. NaN, inf, Längen) — Existenz mechanisch, Suffizienz Ermessen | mechanisch |
| R60 | `RemoteFunction:InvokeClient()` ist verboten | mechanisch |
| R61 | `RemoteFunction` nur bei echter Anfrage-Antwort | Ermessen |
| R62 | `UnreliableRemoteEvent` nur für unkritische Daten | mechanisch |
| R63 | Über die Grenze gehen nur einfache Daten | mechanisch |
| R64 | Jedes Remote hat serverseitig eine Drosselung | mechanisch |
| R65 | Remote-Namen sagen die Richtung | Ermessen |
| R66 | Spielerdaten ausschließlich über ProfileStore | mechanisch |
| R67 | Nur der `DataService` spricht mit ProfileStore | mechanisch |
| R68 | DataStore-Aufrufe außerhalb von ProfileStore in `pcall` | mechanisch |
| R69 | `UpdateAsync`, wenn der neue Wert vom alten abhängt | mechanisch |
| R70 | Im `UpdateAsync`-Callback wird nicht geyieldet | mechanisch |
| R71 | Gespeicherte Daten haben eine Schemaversion | mechanisch |
| R72 | Nichts unbegrenzt Wachsendes speichern | Ermessen |
| R73 | Ranglisten über `OrderedDataStore`, nicht ProfileStore | mechanisch |
| R74 | Bibliotheken nur über `insert_asset` mit fester Asset-ID | mechanisch |
| R75 | Nicht-dauerhafte Verbindungen nachhalten und trennen | mechanisch |
| R76 | Beim Verlassen alles zum Spieler freigeben, auch Tabelleneinträge | mechanisch |
| R77 | Nicht auf `PlayerCharacterDestroyBehavior` verlassen, selbst aufräumen | mechanisch |
| R78 | `Instance:Destroy()`, nie `:Remove()` | mechanisch |
| R79 | Nach `Destroy()` die eigene Referenz auf `nil` setzen | mechanisch |
| R80 | Eigenschaften vor dem Parenten setzen | mechanisch |
| R81 | Attributes statt Value-Objekte | mechanisch |
| R82 | `CollectionService`-Tags statt Namensabfragen | Ermessen |
| R83 | `WaitForChild` nur mit Timeout, `nil`-Fall behandelt | mechanisch |
| R84 | Modultabelle direkt unter dem PUBLIC-Banner | mechanisch |
| R85 | Booleans beginnen mit `is`, `has` oder `can` | mechanisch |
| R86 | Keine auskommentierten Code-Leichen | mechanisch |
| R87 | Ein `TODO` trägt ein Datum | mechanisch |
| R88 | Testcode über `execute_luau` sieht eine frische Modulinstanz | Verfahren |
| R89 | In `Init` nichts verbinden und nichts starten; Yielden nur mit Timeout | mechanisch |
| R90 | Messung und auslösende Aktion in einem `execute_luau` | Verfahren |
| R91 | Signal-Handler prüft angefasste Instanzen erneut (`Deferred`) | mechanisch |
| R92 | Ein lokaler Handler ruft nie die eigene Modultabelle | mechanisch |
| R93 | `script_grep` taugt nicht als alleiniger Nachweis | Verfahren |
| R94 | Kein Interaktionszustand in einer schwachen Tabelle | mechanisch |
| R95 | Zeitmessungen über die Werkzeugkette tragen einen Aufschlag | Verfahren |
| R96 | Hängende Berührung im Emulator legt die Kamera still, lautlos | Verfahren |
| R97 | Zeigereingaben kommen versetzt an — vor der ersten Kompensation in **beiden** Achsen und **gegen die Zielart** (Welt oder GUI) messen, der Wert kann null sein; Gegenprobe pflicht | Verfahren |
| R98 | GUI-Inset nur über `GuiService:GetGuiInset()`, nie als Zahl festschreiben; erst bei Benutzung lesen, nicht in `Init`/`Start`; bildschirmfüllende Fläche braucht `IgnoreGuiInset = true` | mechanisch |
| R99 | Plattformweiche nie an `MouseEnabled`/`TouchEnabled` — nach zuletzt geführter Eingabe entscheiden | mechanisch |
| R100 | Lesende Werkzeuge sehen im Play-Modus nur einen DataModel — „gibt es nicht" kann „falscher Zweig" heißen | Verfahren |
| R101 | Kommentar über fremdes Verhalten ist Vertragskopie — mit dem Vertrag nachziehen oder melden | Ermessen |
| R102 | `multi_edit`: jede Ersetzung sucht im Ergebnis der vorherigen — eindeutig verankern, zurücklesen; zur Kontrolle bereichsweise, zum Verstehen ganz | Verfahren |
| R103 | Sichtbarer Spielertext ist englisch | mechanisch |
| R104 | Bezugspunkt einer Messung nie aus der Messmenge — eine Auswertung ohne mögliches Gegenbeispiel ist keine Prüfung | Verfahren |
| R105 | Restlistenpunkt erledigt → im Bericht melden; Restlistenpunkt beauftragen → vorher am heutigen Code prüfen und die Prüfung datieren. Einträge, die von einer Suche abhängen, tragen ihren Prüfbefehl | Verfahren |
| R106 | Eine Zahl im Kommentar neu gemessen → jede Zahl desselben Absatzes neu messen, oder der stehen gebliebenen ihre Messbedingung anschreiben | Verfahren |
| R107 | Eine ausführliche Begründung steht an **einer** Stelle; andere nennen die Aussage kurz und verweisen namentlich dorthin | Ermessen |
| R108 | Verborgener Spielzustand bleibt bis zur Aufdeckung auf dem Server — nicht in Attributes, Payloads oder unsichtbaren GUIs | mechanisch |
| R109 | Kein Yield zwischen serverseitiger Prüfung und Wirkung — sonst Wiedereintritts-Guard je Spieler und Aktion | mechanisch |
| R110 | Ein `Added`-Signal meldet nur die Zukunft — Bestand nachziehen (`GetPlayers()`, `player.Character`, `GetTagged()`, `GetChildren()`) | mechanisch |
| R111 | Registry-Typ je Seite an einer Stelle statt `{ [string]: any }` — Tippfehler zur Analysezeit | Ermessen |
| R112 | Kein Code unter `StarterGui` — UI-Logik ist ein Controller | mechanisch |
| R113 | `ResetOnSpawn` pro ScreenGui bewusst entschieden, Entscheidung im Controller erkennbar | mechanisch |
| R114 | Der Reviewer erhebt das Remote-Inventar unabhängig aus dem Code | Verfahren |
| R115 | Jede Änderung nennt ihre Evidenz; Evidenzklasse passt zur Änderungsklasse | Verfahren |
| R116 | Kurzlebiger, geteilter Cross-Server-Zustand über `MemoryStore`, nie ProfileStore; MemoryStore ist nicht dauerhaft (TTL) | Ermessen |
| R117 | Dev/Prod trennt ein Umgebungs-Scope per `IsStudio()`, an einer Stelle im DataService in jeden Store | mechanisch |
| R118 | Klang-Eintrag ist eine `{ Id, Volume }`-Table; `Id = 0` heißt kein Klang (still, kein Crash) | mechanisch |
| R119 | Klang nur über den zentralen Director (`SoundDirector`/`MusicController`); kein anderes Modul erzeugt `Sound` oder ruft `:Play()` | mechanisch |
| R120 | Neue Aktion, die einen Klang tragen könnte, bekommt sofort den Play-Aufruf auf einen leeren (`Id = 0`) Eintrag | Ermessen |
| R121 | Bild-Ids zentral in `Data.ImageIds`; `Id = 0` heißt kein Bild; Platzhalter-Icon = `Id = 0`-Eintrag | mechanisch |
| R122 | Toggle-Button mit Icon trägt zwei Icons (an/aus), beide in `Data.ImageIds`; Aktions-Knopf nur eins | mechanisch |

---

## Noch offen

Punkte, über die noch nicht entschieden wurde. Bis zur Klärung sind sie **nicht**
Bestandteil des Standards und dürfen vom Reviewer nicht beanstandet werden.

**Noch zu erarbeiten**
- **Block 4 — Verbote:** was in diesen Projekten nie vorkommen darf.

**Zu verifizieren**
- Luau hat einen neuen Typprüfer bekommen, der genau die Reibung reduzieren sollte, die
  gegen `--!strict` spricht (R29). Ob er inzwischen Standard ist und wie viel er
  wegnimmt, sollte gegen die aktuellen Roblox-Docs geprüft werden — die Entscheidung
  könnte danach anders ausfallen.
- `Workspace.StreamingEnabled` ist im Standard nicht festgehalten — dabei entscheidet
  dieser eine Schalter, was auf dem Client überhaupt existiert und wie viel `WaitForChild`
  Client-Code braucht. Wie beim `AuthorityMode`: einmal prüfen, Stand mit Datum in Block 3
  eintragen. *(Aufgenommen am 9. August 2026.)*
- Die sitzungsgebundenen Messwerte in R95/R97/R98 (−62, +58, 0,28 s) sind Kalibrierdaten,
  keine Normen — und der R97-Nachtrag zeigt, dass Agenten Zahlen aus Blockzitaten
  übernehmen. Vorschlag nach R107: in einen Anhang „Umgebungsbefunde" auslagern, auf den
  die Verfahrensregeln zeigen; der Standard behielte das Verfahren, der Anhang die Zahlen
  samt Datum. *(Aufgenommen am 9. August 2026.)*
