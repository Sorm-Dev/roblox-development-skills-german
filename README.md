# Roblox-Entwicklung mit KI — Standard, Setup und Agenten für Claude Code

Dieses Repository ist ein fertiges Werkzeug-Set für die **KI-gestützte Roblox-Entwicklung**
mit [Claude Code](https://claude.com/claude-code). Es gibt der KI feste Regeln und eine klare
Arbeitsteilung, damit sie Roblox-Code sauber, sicher und immer nach demselben Hausstandard
schreibt — statt bei jedem Projekt von vorne zu erklären, wie man es haben möchte.

Es besteht aus zwei Teilen, die zusammenarbeiten:

- **Zwei Skills** geben der KI das Wissen: ein ausführlicher Coding-Standard und eine
  Schritt-für-Schritt-Anleitung zum Aufsetzen neuer Projekte.
- **Zwei Agenten** teilen die Arbeit auf: ein **Coder**, der den Code schreibt und in Roblox
  Studio testet, und ein **Reviewer**, der diesen Code gegen den Standard prüft, bevor er ins
  Spiel kommt.

So entsteht ein einfacher Ablauf: Du besprichst die Aufgabe mit Claude, der Coder setzt sie um,
der Reviewer prüft sie. Das Ergebnis ist gleichmäßiger Code und weniger Fehler.

## Was ist drin

| Ordner / Datei | Was es ist |
| :--- | :--- |
| `roblox-standards/` | Der **Coding-Standard** (ein Skill). Über 120 nummerierte Regeln zu Architektur, Namen, Kommentaren, Remotes, Datenspeicherung und den Fallstricken der Test-Umgebung. Jede Regel sagt, ob sie hart oder Ermessenssache ist. |
| `roblox-projekt-aufsetzen/` | Die **Anleitung für neue Projekte** (ein Skill). Führt Schritt für Schritt durch das Aufsetzen — vom Veröffentlichen des Place über die Studio-Einstellungen bis zu fertigen Referenz-Modulen. Bringt auch die Arbeitsweise (Lead, Coder, Reviewer) als Vorlage mit. |
| `agents/roblox-coder.md` | Der **Coder-Agent**. Schreibt und testet Roblox-Code in einer laufenden Studio-Instanz. |
| `agents/roblox-reviewer.md` | Der **Reviewer-Agent**. Prüft den Code in mehreren Durchgängen gegen den Standard und gibt ein Urteil ab. |

## Voraussetzungen

- **Claude Code** ist installiert (die CLI oder die App von Anthropic).
- Für die **Agenten** brauchst du zusätzlich eine **Roblox-Studio-MCP-Verbindung**, die Claude
  mit einer laufenden Roblox-Studio-Instanz verbindet. Ohne sie funktionieren die Skills allein
  (die KI kann den Standard lesen), aber die Agenten können keinen Roblox-Code anfassen.

---

## Installation — Schritt für Schritt

Claude Code sucht Skills und Agenten in einem festen Ordner in deinem Benutzerverzeichnis:
`~/.claude/`. Die Installation kopiert die Inhalte dieses Repos einfach dorthin.

> **Windows:** `~/.claude/` ist der Ordner `C:\Users\<DeinName>\.claude\`. In der PowerShell und
> in der Git-Bash kannst du `~` genauso schreiben wie unten gezeigt.

### 1. Repository holen

```bash
git clone https://github.com/Sorgdev/roblox-development-skills.git
cd roblox-development-skills
```

### 2. Die zwei Skills installieren

Kopiere die beiden Skill-Ordner nach `~/.claude/skills/`:

```bash
mkdir -p ~/.claude/skills
cp -r roblox-standards roblox-projekt-aufsetzen ~/.claude/skills/
```

### 3. Die zwei Agenten installieren

Kopiere die beiden Agenten-Dateien nach `~/.claude/agents/`:

```bash
mkdir -p ~/.claude/agents
cp agents/roblox-coder.md agents/roblox-reviewer.md ~/.claude/agents/
```

### 4. Prüfen

Starte Claude Code. Es sollten jetzt verfügbar sein:

- die Skills **`roblox-standards`** und **`roblox-projekt-aufsetzen`**,
- die Agenten **`roblox-coder`** und **`roblox-reviewer`**.

Fertig. Ab hier kann die KI nach dem Hausstandard arbeiten.

> **Wichtig zu den Ordnernamen:** Der Ordner `roblox-standards` muss genau so unter
> `~/.claude/skills/` liegen. Die Agenten laden den Standard über den Pfad
> `~/.claude/skills/roblox-standards/SKILL.md`. Stimmt der Ort oder der Name nicht, finden sie
> ihn nicht.

---

## So arbeitest du damit

Du sprichst mit Claude als **Lead**. Der Lead bespricht die Aufgabe mit dir, schneidet einen
klaren Auftrag zu und gibt ihn an den **Coder** weiter. Der Coder schreibt und testet den Code
in Studio. Danach prüft der **Reviewer** das Ergebnis gegen den Standard und meldet, was noch
zu tun ist. Diese Arbeitsweise ist im Skill `roblox-projekt-aufsetzen` ausführlich beschrieben —
dort liegt auch eine Vorlage, die du als `CLAUDE.md` in dein Projekt legen kannst.
