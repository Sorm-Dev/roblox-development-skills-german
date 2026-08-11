---
name: roblox-projekt-aufsetzen
description: Setzt ein neues Roblox-Projekt nach dem Hausstandard auf — Place veröffentlichen, Studio-Einstellungen, DataModel-Grundgerüst, beide Module-Loader, externe Bibliotheken über insert_asset, Referenzmodule für DataService und InputController, und die Arbeitsregel als CLAUDE.md im Projektordner. Verwenden, wenn ein neues Roblox-Projekt begonnen wird oder ein bestehendes auf den Standard gebracht werden soll.
---

# Neues Roblox-Projekt aufsetzen

Schritt-für-Schritt-Anleitung für ein neues Projekt nach dem Hausstandard. Ergänzt den
Coding-Standard — dort stehen die Regeln, hier steht der Ablauf.

**Der Standard ist als Skill `roblox-standards` verfügbar** (liegt in `~/.claude/skills/`).
Regelnummern (`R31`) verweisen dorthin.

---

## Wer macht was

| Schritt | Ausführung |
| :--- | :--- |
| 1 — Place veröffentlichen | **nur Mensch** |
| 2 — Studio-Einstellungen | **nur Mensch** |
| 3 — DataModel-Grundgerüst | Lead (`execute_luau`), Bestätigung durch den Projektleiter |
| 4 — Module-Loader | `roblox-coder` |
| 5 — Bibliotheken | Lead (`insert_asset`), Asset-ID vom Projektleiter verifiziert |
| 6 — Referenzmodule | `roblox-coder` |
| 7 — `CLAUDE.md` | Lead (Dateisystem) |
| 8 — Erstprüfung | Lead |

**Die Schritte 4 und 6 schreiben Roblox-Code — dafür wird ein `roblox-coder` gespawnt.**
Die Hauptsession fasst Roblox-Code nicht selbst an. Ob danach ein `roblox-reviewer` läuft,
richtet sich nach der Trivialregel in der `CLAUDE.md` aus Schritt 7; beim Aufsetzen sind es
mehrere Dateien mit Verhalten, also **ja**.

**Schritt 1 und 2 kann kein Agent übernehmen.** Beide liegen außerhalb dessen, was über den
Studio-MCP erreichbar ist — Veröffentlichen und die Sicherheitseinstellungen im Game-
Settings-Dialog sind Konto- und Web-Vorgänge, keine DataModel-Operationen. Ein Agent, der
behauptet, sie erledigt zu haben, irrt sich.

**Schritt 3 braucht `execute_luau`**, weil `multi_edit` nur Skripte anlegen kann, keine
`Folder`-Instanzen. Nach der Rechtestruktur heißt das: Lead führt aus, Projektleiter
bestätigt.

---

## Schritt 1 — Place anlegen und veröffentlichen

> **Nur Mensch.** Kein Agent kann das.

Das Place muss **veröffentlicht** sein, bevor irgendetwas mit Daten funktioniert. In einem
unveröffentlichten Place gibt es keine DataStores — der Code ist dann richtig und schlägt
trotzdem fehl.

## Schritt 2 — Studio-Einstellungen

> **Nur Mensch.** Kein Agent kann das.

**`Game Settings → Security → Enable Studio Access to API Services`** einschalten. Ohne
das schlagen alle DataStore-Aufrufe im Studio fehl.

**`Workspace.PlayerCharacterDestroyBehavior` bleibt auf dem Standardwert** (R77). Wir
verlassen uns nicht darauf — aufgeräumt wird im Code, nach R75 und R76.

> Zur Einordnung: Player-Objekte und Charakter-Modelle werden nach dem Verlassen nicht
> automatisch zerstört. Laut Roblox-Doku ist das eine Quelle „sehr erheblicher
> Speicherlecks", wenn über die Serverlaufzeit hunderte Spieler kommen und gehen. Genau
> deshalb ist manuelles Trennen von Verbindungen und Löschen von Tabelleneinträgen keine
> Kür.

> Diese beiden Punkte sind die häufigste Ursache für „der Datencode funktioniert nicht",
> obwohl er korrekt ist. Wer hier hängt, sucht sonst stundenlang im eigenen Code.

## Schritt 3 — DataModel-Grundgerüst

> **Lead mit Bestätigung.** `Folder`-Instanzen lassen sich nicht mit `multi_edit` anlegen —
> das erzeugt nur Skripte. Hierfür ist `execute_luau` nötig.

```
ServerScriptService
  Main                    (Script)
  Services      (Folder)
  ServerData    (Folder)
  Libraries     (Folder)          ← externe Module, NICHT in Services

ReplicatedStorage
  Shared        (Folder)
    Config      (Folder)
    Data        (Folder)
    Types                 (ModuleScript)
    Util        (Folder)
  Remotes       (Folder)

StarterPlayer
  StarterPlayerScripts
    Main                  (LocalScript)
    Controllers (Folder)
```

`Libraries` liegt bewusst **neben** `Services`, nicht darin: der Loader lädt alles aus
`Services` und würde eine Bibliothek sonst als Service in die Registry aufnehmen (R31,
R37).

> **Achtung bei Bestandsprojekten.** Lädt ein Loader aus seinen *eigenen Kindern*
> (`MODULE_FOLDER = script`) statt aus einem Geschwister-Ordner, gibt es diesen Platz
> nicht. Eine per `insert_asset` unter den Loader gelegte Bibliothek landet dann in der
> Registry und bekommt `Init`/`Start` untergeschoben. **Vor der ersten
> Bibliotheksinstallation klären** — entweder die Module in einen eigenen Ordner umziehen
> oder die Bibliothek außerhalb des Loaders ablegen.

## Schritt 4 — Module-Loader

Zwei Loader, einer pro Seite. Sie sind identisch bis auf den Ordner, aus dem sie laden.

### `ServerScriptService.Main` (Script)

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

### `StarterPlayerScripts.Main` (LocalScript)

Derselbe Code, nur die eine Konstante ändert sich:

```lua
local MODULE_FOLDER = script.Parent.Controllers
```

Server-Services und Client-Controller sind **zwei getrennte Registries**. Ein Service
taucht nie in der Client-Registry auf.

## Schritt 5 — Externe Bibliotheken installieren

> **Lead mit Bestätigung.** Die Asset-ID muss vorher vom Projektleiter verifiziert sein.

> **Regel: Bibliotheken werden ausschließlich über `insert_asset` mit einer hier
> festgeschriebenen Asset-ID installiert. Kein Agent kopiert fremden Quellcode in ein
> Projekt.**

Grund: `multi_edit` legt ein Skript an, indem der Agent den kompletten Inhalt als String
ausgibt — bei tausend Zeilen fremdem Code ist das keine Kopie, sondern eine **Abschrift**.
Ein stiller Abschriftfehler in der Datenschicht ist genau die Fehlerklasse, gegen die die
Bibliothek eingesetzt wird, und der Reviewer kann ihn nicht finden. Bei `insert_asset`
kopiert die Engine, nicht das Modell.

Nebenwirkungen, die alle erwünscht sind: jedes Projekt bekommt exakt dasselbe, die Version
ist festgenagelt (Updates werden zu einer bewussten Entscheidung), und die Prüfung des
Urhebers passiert genau einmal — hier.

### Bibliotheksliste

| Bibliothek | Zweck | Ziel | Asset-ID |
| :--- | :--- | :--- | :--- |
| ProfileStore | Spielerdaten, Session-Locking | `ServerScriptService.Libraries` | `109379033046155` |

```
insert_asset(assetId: "109379033046155", assetName: "ProfileStore",
             parentPath: "game.ServerScriptService.Libraries")
```

> **Eingetragen 10.8.2026:** Asset-ID `109379033046155`, vom Projektleiter über den offiziellen
> Creator-Store-Link geliefert (`create.roblox.com/store/asset/109379033046155/ProfileStore`);
> Creator loleris / MadStudio (der ProfileStore-Autor) per Recherche bestätigt. Beim
> `insert_asset` in Studio dennoch den Creator im Toolbox-Eintrag gegenprüfen und die Version
> notieren — eine Asset-ID ist kein Content-Hash (R74), der Creator kann das Asset aktualisieren.

**ProfileStore ist der Nachfolger von ProfileService.** Wo in älteren Notizen
„ProfileService" steht, ist ProfileStore gemeint.

## Schritt 6 — Referenzmodule

Drei Module, die fast jedes Projekt braucht. Alle sind standardkonform geschrieben
und als Ausgangspunkt gedacht, nicht als unveränderliche Vorlage.

### 6a — DataService

Referenzmodul nach R66–R73. Liegt unter `ServerScriptService.Services.DataService` und ist
das **einzige** Modul, das ProfileStore je sieht (R67).

```lua
--- Loads, caches and saves player profiles through ProfileStore.


---------------
--- IMPORTS ---
---------------

local Players = game:GetService("Players")
local ServerScriptService = game:GetService("ServerScriptService")


local ProfileStore = require(ServerScriptService.Libraries.ProfileStore)


-------------
--- TYPES ---
-------------

--- The shape of a stored player profile
export type PlayerData = {
	SchemaVersion: number,
	Cash: number,
	Items: { string },
}


-------------
--- LOCAL ---
-------------

--- Constants
local STORE_NAME = "PlayerStore"
local SCHEMA_VERSION = 1
local PROFILE_TEMPLATE: PlayerData = {
	SchemaVersion = SCHEMA_VERSION,
	Cash = 0,
	Items = {},
}


--- Active store and the profiles cached on this server
local _store = ProfileStore.New(STORE_NAME, PROFILE_TEMPLATE)
local _profiles = {}


--- Bindables backing the public signals
local _profileLoadedBindable = Instance.new("BindableEvent")


--- Brings an older profile up to the current schema version
local function _migrate(data: PlayerData)
	if data.SchemaVersion == SCHEMA_VERSION then
		return
	end

	data.SchemaVersion = SCHEMA_VERSION
end


--- Starts a session and caches the profile for one player
local function _loadProfile(player: Player)
	local profile = _store:StartSessionAsync(`{player.UserId}`, {
		Cancel = function()
			return player.Parent ~= Players
		end,
	})

	if profile == nil then
		player:Kick("Profile load failed - please rejoin")
		return
	end

	profile:AddUserId(player.UserId)
	profile:Reconcile()
	_migrate(profile.Data)

	profile.OnSessionEnd:Connect(function()
		_profiles[player] = nil
		player:Kick("Profile session ended - please rejoin")
	end)

	if player.Parent ~= Players then
		profile:EndSession()
		return
	end

	_profiles[player] = profile
	_profileLoadedBindable:Fire(player)
end


--- Ends the session so another server can take over
local function _releaseProfile(player: Player)
	local profile = _profiles[player]

	if profile ~= nil then
		profile:EndSession()
	end
end


--------------
--- PUBLIC ---
--------------

local DataService = {}


--- Fires with the player once their profile is loaded and ready
DataService.ProfileLoaded = _profileLoadedBindable.Event


--- Returns the player's data, or nil while the profile is still loading
function DataService.GetData(player: Player): PlayerData?
	local profile = _profiles[player]

	if profile == nil then
		return nil
	end

	return profile.Data
end


--- Connects player lifecycle handling
function DataService.Start()
	for _, player in Players:GetPlayers() do
		task.spawn(_loadProfile, player)
	end

	Players.PlayerAdded:Connect(_loadProfile)
	Players.PlayerRemoving:Connect(_releaseProfile)
end


return DataService
```

### Was man daran verstanden haben muss

- **`PROFILE_TEMPLATE`** ist die Standardstruktur für neue Spieler.
- **`Reconcile()`** ergänzt fehlende Felder aus dem Template — so bekommen Bestandsspieler
  neue Felder, wenn das Template wächst.
- **`_migrate` und `SchemaVersion` (R71)** decken den anderen Fall ab: ein Feld existiert,
  hat aber seine Bedeutung geändert. Reconcile hilft da nicht.
- **`AddUserId()`** ist für die DSGVO-Löschanfragen von Roblox nötig.
- **`OnSessionEnd`** feuert, wenn ein anderer Server die Session übernimmt. Der Spieler
  muss dann raus — sonst spielt er mit Daten weiter, die ihm nicht mehr gehören.
- **`task.spawn` beim Nachladen** vorhandener Spieler, weil `StartSessionAsync` yieldet.
- **`GetData` gibt `nil` zurück**, solange das Profil lädt. Aufrufer müssen das behandeln —
  deshalb gibt es zusätzlich das Signal `ProfileLoaded`.

### 6b — InputController

Referenzmodul nach R49–R57. Liegt unter `StarterPlayerScripts.Controllers.InputController`
und ist das **einzige** Modul, das je eine Input-API berührt (R49).

> **Die Aktionen sind spielspezifisch, die Struktur ist es nicht.** `Jump` und `Interact`
> unten sind Platzhalter — ersetzt sie durch die Aktionen des jeweiligen Spiels. Was
> bleibt: Konstanten für die Namen, ein privates Bindable pro Aktion, ein Handler mit
> explizitem Rückgabewert, und ein Paar aus `BindGameplay` und `UnbindGameplay`.

```lua
--- Translates raw input into game signals; the only module that touches an input API.


---------------
--- IMPORTS ---
---------------

local ContextActionService = game:GetService("ContextActionService")


-------------
--- LOCAL ---
-------------

--- Constants
local ACTION_JUMP = "Jump"
local ACTION_INTERACT = "Interact"


--- Bindables backing the public signals
local _jumpBindable = Instance.new("BindableEvent")
local _interactBindable = Instance.new("BindableEvent")


--- Handles the jump action
local function _onJump(_, inputState: Enum.UserInputState): Enum.ContextActionResult
	if inputState == Enum.UserInputState.Begin then
		_jumpBindable:Fire()
	end

	return Enum.ContextActionResult.Sink
end


--- Handles the interact action
local function _onInteract(_, inputState: Enum.UserInputState): Enum.ContextActionResult
	if inputState == Enum.UserInputState.Begin then
		_interactBindable:Fire()
	end

	return Enum.ContextActionResult.Sink
end


--------------
--- PUBLIC ---
--------------

local InputController = {}


--- Fires when the player triggers the jump action
InputController.Jump = _jumpBindable.Event


--- Fires when the player triggers the interact action
InputController.Interact = _interactBindable.Event


--- Binds the gameplay actions across keyboard, gamepad and touch
function InputController.BindGameplay()
	ContextActionService:BindAction(ACTION_JUMP, _onJump, true,
		Enum.KeyCode.Space, Enum.KeyCode.ButtonA)

	ContextActionService:BindAction(ACTION_INTERACT, _onInteract, true,
		Enum.KeyCode.E, Enum.KeyCode.ButtonX)
end


--- Releases the gameplay actions, for example while a menu is open
function InputController.UnbindGameplay()
	ContextActionService:UnbindAction(ACTION_JUMP)
	ContextActionService:UnbindAction(ACTION_INTERACT)
end


--- Starts with gameplay input active
function InputController.Start()
	InputController.BindGameplay()
end


return InputController
```

#### Was man daran verstanden haben muss

- **Das dritte Argument von `BindAction` ist `createTouchButton`.** Steht es auf `true`,
  erzeugt die Engine den Touch-Button selbst — das ist die halbe Miete für R52. Die
  Gamepad-KeyCode kommt in derselben Zeile mit dazu.
- **`_` als erster Parameter** ist der Aktionsname, den der Handler nicht braucht. Nach
  R19 ist das die korrekte Schreibweise für einen bewusst ignorierten Wert.
- **`Sink` statt gar nichts** (R53): ohne Rückgabe läuft die Eingabe an tiefer gebundene
  Aktionen weiter.
- **`BindGameplay` und `UnbindGameplay` als Paar** (R54, R55). Ein Menü-Controller ruft
  `UnbindGameplay` beim Öffnen und `BindGameplay` beim Schließen — statt in jedem Handler
  ein Flag abzufragen.
- **Nur `.Event` ist öffentlich**, das `BindableEvent` bleibt privat (R56). Andere
  Controller holen sich den `InputController` injiziert und verbinden sich in `Start()`
  auf `_inputController.Jump`.

### 6c — SoundDirector

Referenzmodul nach R118–R120. Liegt unter `StarterPlayerScripts.Controllers.SoundDirector`
und ist das **einzige** Modul, das je eine `Sound`-Instanz erzeugt oder `:Play()` ruft (R119).
Andere Module lösen Klang nur über die öffentliche `Play`-API aus.

> **Die Gruppen und Schlüssel sind spielspezifisch, das Muster ist es nicht.** `Ui` und
> `World` unten sind Platzhalter. Was bleibt: die `{ Id, Volume }`-Nachschlagetabelle, der
> **wiederverwendete Stimmen-Pool** statt `Instance.new` je Klang, das Frame-Budget, die
> **Volume × Dämpfung**-Kombination an einer Stelle, und die scharfe Trennung von *gewollter
> Stille* (`Id = 0`) und *Fehler* (fehlender Schlüssel wirft). Die spielspezifischen
> Play-Funktionen sind bewusst **weggelassen** — sie zeigen das Muster nicht klarer,
> nur länger.

```lua
--- Turns game events into sound; the only module that ever starts a Sound.


---------------
--- IMPORTS ---
---------------

local RunService = game:GetService("RunService")
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")


local SoundIds = require(ReplicatedStorage.Shared.Data.SoundIds)


-------------
--- TYPES ---
-------------

--- One entry in Data.SoundIds: the asset id and the volume it plays at
export type SoundEntry = {
	Id: number,
	Volume: number,
}


-------------
--- LOCAL ---
-------------

--- Constants. A grown project usually moves these into Config; they sit here so
--- the reference reads on its own.
local POOL_SIZE = 8 -- reused Sound voices; never Instance.new per play
local MAX_STARTS_PER_FRAME = 4 -- ceiling on how many sounds may begin in one frame
local OTHER_PLAYER_INTENSITY = 0.4 -- how far another player's action is damped
local EARSHOT = 120 -- studs; a positioned sound past this is not worth starting
local ASSET_PREFIX = "rbxassetid://"
local SOUND_FOLDER_NAME = "ClientSound"
local VOICE_NAME_PREFIX = "Voice"
local DEFAULT_VOLUME = 1


local EARSHOT_SQUARED = EARSHOT * EARSHOT


--- The reused pool, held by index, and where the rotation currently stands
local _voices: { Sound } = {}
local _nextVoice = 1


--- What is left of this frame's start budget
local _startBudget = MAX_STARTS_PER_FRAME


--- The entry filed for one key, read afresh on every start so an id typed into
--- Data.SoundIds takes effect from the next play with no code change. A missing
--- entry is a typo at the call site and THROWS; an entry whose id is 0 is one
--- nobody has filed yet and is skipped in silence by the caller. A silent skip
--- for a typo would make the two look alike from the outside - hence the throw.
local function _entryFor(group: string, key: string): SoundEntry
	local entries = SoundIds[group]
	local entry = entries and entries[key]

	if type(entry) ~= "table" or type(entry.Id) ~= "number" then
		error(string.format("SoundDirector: Data.SoundIds has no entry %s.%s", group, key))
	end

	return entry
end


--- Hands out the next voice in a fixed rotation - not a search for a free one,
--- because that would have to ask every Sound whether it is playing and the
--- answer moves underneath the loop.
local function _takeVoice(): Sound
	local voice = _voices[_nextVoice]
	_nextVoice += 1

	if _nextVoice > POOL_SIZE then
		_nextVoice = 1
	end

	return voice
end


--- Takes one start out of this frame's budget, or reports it spent
local function _takeStartSlot(): boolean
	if _startBudget <= 0 then
		return false
	end

	_startBudget -= 1

	return true
end


--- How loud a sound plays: its filed volume, damped when the action is somebody
--- else's so a crowd never buries the answer to this player's own hand. This is
--- the ONE place a damping meets the per-sound volume - a project with more of
--- them (a master slider, a menu duck) folds them in here too.
local function _volumeFor(volume: number, isOwn: boolean): number
	return if isOwn then volume else volume * OTHER_PLAYER_INTENSITY
end


--- Starts one sound, or drops it. The gates run in a fixed order: an unfiled id
--- costs no start slot, so the frame budget is only ever spent on a sound that
--- will really play.
local function _playNow(group: string, key: string, isOwn: boolean)
	local entry = _entryFor(group, key)

	if entry.Id == 0 then
		return
	end

	if not _takeStartSlot() then
		return
	end

	local voice = _takeVoice()
	voice.SoundId = ASSET_PREFIX .. entry.Id
	voice.Volume = _volumeFor(entry.Volume, isOwn)
	voice:Play()
end


--- Whether a world position is close enough to the listener to be worth hearing
local function _isWithinEarshot(position: Vector3): boolean
	local camera = Workspace.CurrentCamera

	if camera == nil then
		return false
	end

	local offset = camera.CFrame.Position - position

	return offset:Dot(offset) <= EARSHOT_SQUARED
end


--- One frame: the start budget turns over. Granting it here, where the frame
--- step turns the frame over, and never lazily on first use, keeps an event
--- delivered just before the step from sharing an account with one just after.
local function _step()
	_startBudget = MAX_STARTS_PER_FRAME
end


--- Builds the pool. Every voice starts without an id, so nothing here ever asks
--- the engine for asset 0; a start writes the id it needs and the next one
--- writes over it. The voices hang under SoundService and not on a part, so they
--- play flat. Properties are set before the folder is parented (R80).
local function _buildVoices()
	local folder = Instance.new("Folder")
	folder.Name = SOUND_FOLDER_NAME

	for voiceIndex = 1, POOL_SIZE do
		local voice = Instance.new("Sound")
		voice.Name = VOICE_NAME_PREFIX .. voiceIndex
		voice.Volume = DEFAULT_VOLUME
		voice.Parent = folder
		_voices[voiceIndex] = voice
	end

	folder.Parent = SoundService
end


--------------
--- PUBLIC ---
--------------

local SoundDirector = {}


--- Plays a sound that belongs nowhere in particular: an answer to an input, a
--- round event, a menu event. Always this player's own, always full volume.
function SoundDirector.Play(group: string, key: string)
	_playNow(group, key, true)
end


--- Plays a sound that happens at a place, if that place is close enough to hear,
--- damped when the action is another player's. The position decides only whether
--- to start at all; the voices still play flat. A project that wants true 3D
--- falloff parents the voice to an Attachment at `position` instead.
function SoundDirector.PlayAt(group: string, key: string, position: Vector3, isOwn: boolean)
	if not _isWithinEarshot(position) then
		return
	end

	_playNow(group, key, isOwn)
end


--- Builds the voice pool. It is this module's own state, so it is built in Init
--- and not Start: a sibling's Start may run first and already ask for a sound,
--- and the order inside a phase is undefined (R34). Nothing is connected here (R89).
function SoundDirector.Init()
	_buildVoices()
end


--- Turns the frame budget over every Heartbeat. The connection is deliberately
--- permanent - it lives exactly as long as the client does.
function SoundDirector.Start()
	RunService.Heartbeat:Connect(_step)
end


return SoundDirector
```

Und das dazugehörige `Data.SoundIds` im `{ Id, Volume }`-Format — wenige Einträge, einer
davon `Id = 0` als „noch nicht vertont". Liegt unter `ReplicatedStorage.Shared.Data.SoundIds`:

```lua
--- Every sound asset id in one place. Each entry is a table of an asset id and
--- the volume it plays at, so one sound's loudness can be tuned on its own
--- without touching code. An Id of 0 means none has been filed yet: the
--- SoundDirector skips those silently (R118), so a missing sound can never block
--- the game, and their volume is only what they will play at once an id is filed.
---
--- Filing an id here is the ONLY change a sound needs - not one line of code
--- moves with it. The ids below are placeholders; replace them with your own
--- uploads.
local SoundIds = {}


--- Interface sounds
SoundIds.Ui = {
	ButtonClick = { Id = 9118823103, Volume = 1 }, -- Beispiel-Id, ersetzen
	MenuOpen = { Id = 9125402735, Volume = 0.8 }, -- Beispiel-Id, ersetzen
	MenuClose = { Id = 0, Volume = 0.8 }, -- noch nicht vertont
}


--- Things that happen out in the world
SoundIds.World = {
	Pickup = { Id = 9043802718, Volume = 1 }, -- Beispiel-Id, ersetzen
	Spawn = { Id = 0, Volume = 1 }, -- noch nicht vertont
}


--- table.freeze is shallow, so every entry, every group and the whole table need
--- their own call now that an entry is itself a table (R7/R8).
for _, group in SoundIds do
	for _, entry in group do
		table.freeze(entry)
	end
	table.freeze(group)
end

table.freeze(SoundIds)

return SoundIds
```

#### Was man daran verstanden haben muss

- **`{ Id, Volume }` und `Id = 0` (R118).** Der Eintrag trägt Asset-Id und Lautstärke in
  *einer* Table. `Id = 0` heißt „noch nicht vertont": die Stelle bleibt **still statt zu
  crashen**, ein unbefüllter Klang bricht das Spiel nie. Tief eingefroren (R7/R8), Schlüssel
  sind stabile Strings.
- **Fehlender Schlüssel wirft (R119).** `_entryFor` unterscheidet damit *gewollte Stille*
  (`Id = 0`, still) von einem *Tippfehler* am Aufrufort (kein Eintrag, `error`) — sonst
  sähen beide von außen gleich aus, und ein vertippter Schlüssel liefe als Stille durch.
- **Wiederverwendeter Stimmen-Pool (R119).** `_buildVoices` legt den Pool einmal an,
  `_takeVoice` rotiert fest hindurch. Kein `Instance.new` je Klang, keine Suche nach einer
  freien Stimme — die Antwort auf „spielt gerade?" bewegt sich unter der Schleife.
- **Frame-Budget.** `MAX_STARTS_PER_FRAME` deckelt, wie viele Klänge *ein* Frame starten
  darf; `_step` teilt das Budget an genau einer Stelle neu aus, wo der Frame umschlägt.
- **`_volumeFor` — die eine Stelle für Dämpfung.** Per-Klang-`Volume` mal Dämpfung
  (`OtherPlayerIntensity` für fremde Aktionen). Jede weitere Dämpfung — Master-Regler,
  Menü-Duck — kommt hierher und nirgends sonst.
- **Eintrag wird nirgends gehalten**, sondern bei jedem Start frisch gelesen — genau das
  lässt eine nachgetragene Id ohne Codeänderung wirken (die Grundlage von R120).
- **`Init` baut den Pool, `Start` verbindet** (R89/R34). Der Pool ist eigener Zustand und
  muss stehen, bevor irgendein Geschwister-`Start` den ersten Klang anfordert.

## Schritt 7 — Die Arbeitsregel als `CLAUDE.md`

**Das ist der Schritt, der das Projekt an das Team anschließt.** `CLAUDE.md` ist der einzige
Kanal, den jede Session garantiert lädt — Regeln, die nur hier stehen, gelten wirklich.

Schreib die folgende Datei in den Projektordner. `<PROJEKTNAME>` ersetzen, sonst wörtlich
übernehmen:

````markdown
# <PROJEKTNAME> — Arbeitsweise

Roblox-Projekt. Der Code lebt in der `.rbxl`-Datei in Roblox Studio, **nicht auf der
Festplatte**, und ist nur über die `mcp__Roblox_Studio__*`-Werkzeuge erreichbar. Kein Rojo.

---

## Die Hauptsession schreibt keinen Roblox-Code

**Ausnahmslos.** Kein `multi_edit`, kein `execute_luau` mit `datamodel_type: "Edit"`, keine
Änderung an einem Skript in Studio — auch keine Einzeiler, auch keine Kommentare.

Wenn Code geschrieben oder geändert werden muss, wird ein **`roblox-coder`** gespawnt.

Die Hauptsession ist der **Lead**: sie spricht mit dem Projektleiter, schneidet Aufträge zu,
delegiert, prüft die Berichte gegeneinander und trifft Entscheidungen, die über einen
einzelnen Auftrag hinausgehen.

## Wann ein Review nötig ist — und wann nicht

**Kein Review, wenn ausschließlich Kommentartext geändert wurde.** Egal wie viele Dateien,
egal wie viele Zeilen. Ein Lauf, der Kommentare berichtigt, kann das Spiel nicht kaputt
machen.

**Sobald eine Codezeile entsteht, fällt oder sich ändert: Review.** Auch wenn es nach nichts
aussieht — ein gelöschtes `Config`-Feld ist eine Codeänderung, eine umbenannte Konstante auch.

Das ist die ganze Regel. Sie ist scharf, weil „ändert das Verhalten?" eine Auslegungsfrage
wäre und „ist es nur Kommentar?" keine.

**Und der Review skaliert mit dem Lauf.** Ein neues Modul mit einer Remote-Grenze verdient
alle fünf Durchgänge; drei geänderte Zeilen in einer bekannten Datei verdienen einen kurzen,
gezielten Blick. Der Lead sagt im Auftrag, welche Tiefe er will.

## Aufträge kurz halten

Ein Auftrag nennt **das Ziel, die Grenzen und die Fallstricke, die dieser Lauf treffen wird**
— nicht alles, was über das Projekt bekannt ist. Der Coder liest den Standard selbst und
findet die Fundstellen selbst; was er nicht selbst findet, ist die Absicht dahinter.

Lange Aufträge kosten doppelt: beim Schreiben und beim Bericht, weil der Coder auf alles
eingeht, was dasteht.

## Der Ablauf

1. **Lead** bespricht mit dem Projektleiter und schneidet den Auftrag zu
2. **`roblox-coder`** setzt um und testet
3. **`roblox-reviewer`** prüft — sofern der Lauf Verhalten geändert hat, siehe oben
4. Bei Auflagen: **derselbe Coder** wird per `SendMessage` fortgesetzt, nicht ein neuer —
   er weiß dann noch, warum er was gebaut hat
5. **Nach zwei Runden ist Schluss.** Bleiben BLOCKER offen, entscheidet der Projektleiter

**Nicht in Studio schreiben, während ein Lauf läuft** — wegen der Kollision in derselben
Datei. Der Lead sagt an, wann etwas läuft und wann es durch ist.

### Kein Abbild des Codes anlegen

Es liegt nahe, die Skripte per `script_read` in ein Verzeichnis zu schreiben, damit der
Reviewer einen Diff hat — ohne Dateien auf der Platte gibt es ja kein `git diff`. **Tu es
nicht.** Genau das wurde ein halbes Jahr lang gemacht und am 7. August 2026 abgeschafft:

Ein Nachzug von neun Dateien kostete zuletzt rund **280.000 Tokens** — mehr als mancher
Coder-Lauf —, weil jede Datei vollständig gelesen und vollständig zurückgeschrieben wird. An
einem sehr produktiven Tag mit einem Dutzend Läufen hat der Diff **keinen einzigen Fund**
hervorgebracht; alle Funde kamen aus dem Lesen des Codes selbst.

Das Abbild sollte verhindern, dass ein Coder mehr ändert, als er meldet. Dieses Risiko hat
sich nie gezeigt.

**Stattdessen:** Der Reviewer liest den Ist-Stand und hält ihn gegen Auftrag und Coder-Bericht.
Der Coder nennt im Bericht den **alten Wortlaut**, wo er eine Begründung oder eine Zahl
ersetzt — das ist die Bringschuld, die den Diff ersetzt.

Der Rückweg im Notfall ist Roblox' eigene Versionshistorie oder die zuletzt gespeicherte
`.rbxl`.

## Der Standard

Skill `roblox-standards` — projektübergreifend (liegt in `~/.claude/skills/`).

Beide Agenten laden ihn selbst; ihre Definitionen tragen den Pfad. Der **Regelindex am
Ende** fasst jede Regel als `R<n>` zusammen und sagt, ob sie *mechanisch* oder *Ermessen*
ist.

**Änderungen am Standard entscheidet der Projektleiter.** Schlägt ein Agent eine neue Regel
vor, formuliert er sie — eingetragen wird sie vom Lead, nach Rücksprache.

## Die Restliste

Jedes Projekt führt ein Dokument, in dem Läufe und Reviews sammeln, was offen blieb — hier
`RESTLISTE.md`. Aus ihm werden später Aufträge geschnitten, und genau deshalb ist ein falsch
gebliebener Eintrag **teurer als gar keiner**: Er beauftragt Arbeit, die es nicht mehr gibt,
und verbraucht das Vertrauen in alle Einträge daneben.

**R105 regelt die beiden Pflichten:** Wer einen Punkt erledigt — auch nebenbei, auch ungefragt
—, sagt es im Bericht; eintragen tut es der Lead. Und wer aus der Liste beauftragt, prüft
jeden Punkt am heutigen Code, bevor er ihn in einen Auftrag schreibt. Eine Prüfung, die einen
Punkt bestätigt, wird datiert.

**Und ein Kniff, der das Nachprüfen fast umsonst macht — von Anfang an mitschreiben, später
nachzurüsten kostet ein Vielfaches:** Ein Eintrag, dessen Wahrheit von einer Suche abhängt,
trägt seinen **Prüfbefehl** mit. Statt „`FxCullDistance` ist ungelesen" schreibt die Liste
*„ungelesen — Beleg: `script_grep FxCullDistance`, 2 Treffer, beide Definition und Kommentar,
6.8.2026."* Der nächste Lauf führt den Befehl aus, statt den Punkt zu untersuchen.

**Zwei Dinge, die sich in einem gewachsenen Projekt als Fehler herausgestellt haben** — mach
sie von Anfang an anders:

- **Nicht nach Herkunftslauf gliedern, sondern nach Sachzusammenhang.** Eine Chronik liest
  sich beim Schreiben natürlich und beim Auftragschneiden furchtbar: Sie verteilt genau die
  Punkte über das ganze Dokument, die ein Lauf zusammen erledigen würde.
- **Den Abschlussbericht eines Umbaus nicht in dieselbe Datei legen.** Er ist fertig und
  ändert sich nie mehr; die Restliste ändert sich ständig. Wer sie zum ersten Mal öffnet,
  soll nicht erst ein Viertel Geschichte lesen.
````

## Schritt 8 — Erste Prüfung

Einmal `roblox-reviewer` über den frischen Bestand laufen lassen. Das prüft
gleichzeitig zwei Dinge: ob das Grundgerüst dem Standard entspricht, und ob die
Agenten-Kette in diesem Projekt überhaupt funktioniert — besser jetzt herausfinden als beim
ersten inhaltlichen Auftrag.

---

## Checkliste

- [ ] **(Mensch)** Place angelegt und **veröffentlicht**
- [ ] **(Mensch)** `Enable Studio Access to API Services` aktiviert
- [ ] (Lead) DataModel-Grundgerüst angelegt, `Libraries` neben `Services`
- [ ] (`roblox-coder`) `Main` (Script) in `ServerScriptService`
- [ ] (`roblox-coder`) `Main` (LocalScript) in `StarterPlayerScripts`
- [ ] (Lead) ProfileStore über `insert_asset` nach `Libraries` installiert
- [ ] (`roblox-coder`) `DataService` in `Services`
- [ ] (`roblox-coder`) `InputController` in `Controllers`
- [ ] (`roblox-coder`) `SoundDirector` in `Controllers`, `SoundIds` in `Shared.Data`
- [ ] (`roblox-reviewer`) Grundgerüst geprüft
- [ ] (Lead) `CLAUDE.md` im Projektordner

Die ersten beiden Haken kann nur der Projektleiter setzen. Alles darunter ist
automatisierbar, sobald sie gesetzt sind.

**Die `CLAUDE.md` schreibt der Lead selbst** — sie ist kein Roblox-Code, und sie muss stehen,
bevor die Regel, die sie enthält, greifen kann.
