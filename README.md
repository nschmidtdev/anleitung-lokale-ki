# Lokale KI selbst betreiben — das Handbuch

> **TL;DR:** Ein eigenes LLM schnell genug für **agentisches Arbeiten** (Coding-Agent,
> Dateizugriff, Tools) ist auf Consumer-Hardware realistisch — wenn man die richtigen
> Stellschrauben kennt. Dieses Repo fasst zusammen, was nach zwei unabhängigen Messreihen
> wirklich zählt — und welche „offensichtlichen“ Dinger nichts bringen.

**Stand: November 2026** · [Benchmark-Rohdaten](docs/benchmarks.md) ·
[Troubleshooting](docs/troubleshooting.md) · [Beiträge willkommen](CONTRIBUTING.md)

---

## Die Messbasis

Alle Aussagen stammen aus **zwei unabhängigen Systemen**, deren Messwerte getrennt
ausgewiesen sind. Wo sich beide einig sind, gilt die Aussage fast immer allgemein.
Wo sie abweichen, zeigt der Label, wer's gemessen hat — **Faustregeln, die nur auf
einem System gelten, sind keine.**

| | System A | System B |
|---|---|---|
| Typ | Consumer-Desktop | Server |
| GPU | RTX 2080 8 GB (Turing, CUDA) | 24 GB Intel Arc (SYCL/oneAPI ≥ 2026.1) |
| RAM | Ryzen 2700, 32 GB DDR4-2133 (~25 GB/s) | Xeon, Quad-Channel (~40 GB/s Read) |
| Speicher | NVMe lokal | NAS mit Parity/Mover-Jobs |
| llama.cpp | b12360 | wechselnd (Build-Frische getestet) |

---

## 1. Lohnt sich das überhaupt?

**Ja, wenn:** Datenschutz zählt (Code, private Daten), du offline arbeiten willst,
Massen-Queries gratis brauchst, oder ein Coding-Agent stundenlang laufen soll, ohne
dass Token-Kosten das Limit setzen. Realbeispiel: 314k Output-Tokens in einer
Lang-Session = **0,00 €** (System A).

**Nein, wenn:** du Top-Qualität bei schwerem Reasoning brauchst. Lokale Modelle der
4–35B-Klasse liegen unterhalb der großen Cloud-Modelle; für Recherche/Design gut, für
hartes Toolchain-Debugging mittel. **Mische: lokal für Masse, Cloud für die harten 5 %.**

## 2. Voraussetzungen

| Komponente | Minimum | Komfort | Warum |
|---|---|---|---|
| GPU VRAM | 8 GB (RTX 2080/3060) | 16–24 GB | Modell bis ~6,5 GB Dateigröße läuft komplett auf GPU = volle Geschwindigkeit |
| RAM | 16 GB | 32 GB | Größere Modelle (Hybrid) brauchen den Rest — Faustregel: Dateigröße + 2–4 GB |
| RAM-Bandbreite | DDR4-2400 | DDR4-3200+/DDR5 bzw. Quad-Channel | Entscheidet Hybrid-Tempo **mehr als die CPU** (A: ~25 GB/s, B: ~40 GB/s) |
| CPU | 6 Kerne | 8+ Kerne | überwiegend Anlieferung; MoE aktiviert nur wenige Params pro Token |
| Plattform | NVIDIA (CUDA) | — | AMD/Intel gehen auch — SYCL ist ernst zu nehmen, aber CUDA ist der gepflasterte Weg |
| Apple | — | M-Serie mit 16+ GB unified | llama.cpp und MLX laufen nativ, MTP teils nur dort |

**Wichtigste Erkenntnis vorab:** Alles > 6–7 GB Dateigröße wird ein *Hybrid* (Schichten
auf CPU), und dann ist die **RAM-Bandbreite dein Geschwindigkeits-Limit** — nicht die GPU.
Beide Systeme bestätigen: `-lm none` (RAM-Load statt mmap) brachte System A +67 %,
System B (mehr Bandbreite) +80 % mit derselben Stellschraube.

## 3. Software: die drei Wege

1. **llama.cpp direkt** (empfohlen für Speed): `llama-server` + GGUF-Modelle,
   OpenAI-kompatible HTTP-API. Volle Kontrolle über alle Tuning-Flags, vorderste Front
   bei neuen Techniken (MTP, MoE-Cache) und Backends (CUDA, SYCL, Vulkan).
2. **Ollama** (einfachste Route): `ollama run …` — fertig. Aber: eigener Build, neue
   Features kommen spät oder nie. Gut zum Anfangen, nicht für Speed-Tuning.
3. **LM Studio** (GUI): komfortabel, kennt viele Optionen — aber nicht alle Flags
   (Stand 11/2026: kein MoE-Hot-Expert-Cache, kein Single-File-MTP).

**Quant-Sweet-Spot:** Q4_K_M / UD-Q4_K_XL. Q5/Q6 nur mit VRAM-Überhang, IQ3/IQ2 nur
für sehr große Modelle mit viel RAM. Bei unsloth-Quants ist „UD" meist die beste Q4-Wahl.

## 4. Modelle: Größenregeln statt Namen (Namen altern schnell)

| Deine VRAM | Modell-Klasse | Tempo, grob |
|---|---|---|
| 8 GB | 4–10B dense oder MoE mit ~3B aktiv | 60–110 tok/s VRAM-only, ~30 tok/s hybrid |
| 12–16 GB | 14–32B dense, 30B-A3B MoE | 30–60 tok/s |
| 24 GB | 32B dense quantisiert, 35B-A3B komplett | 50+ tok/s (B: 59 tok/s bei 43k gemessen) |
| 48 GB+ | 70B-Klasse, große MoE | 20–40 tok/s |

Praxis-Regeln:
- **MoE mit kleinen aktiven Params** (35B-A3B = 3,3B aktiv) gibt viel Qualität pro
  Hybrid-Byte. Dense derselben Größe ist hybrid deutlich langsamer.
- **Coder-Finetunes** lohnen sich für Coding-Agenten; sonst reicht ein Allround-Modell.
- **Ein Modell pro Use-Case** — und der Modellwechsel gehört nicht in den
  Container-Neustart (siehe Kap. 6b, llama-swap).

**Unser bewährtes Trio** (Speed-Coder / Präzision / Balance — mit Hugging-Face-Links,
gemessenen Werten und der kompletten llama-swap-Config als Copy-Paste-Basis):
[docs/models.md](docs/models.md)

## 5. Die Stellschrauben, die wirklich was bringen (gemessen)

| Maßnahme | Effekt | Flag(s) |
|---|---|---|
| KV-Cache quantisieren | fast verlustfrei, halbiert KV — **aber Typ abtesten**: q8_0 hat oft einen Schnell-Kernel; q5_1 läuft „flach“ über alle Tiefen; q5_0 kann katastrophal sein (6,7 statt 60 tok/s, B) | `-ctk/-ctv` pro Backend messen |
| MTP-Speculation (Ein-Datei-MTP-Heads) | +34 % bei 9B, bis 112 tok/s bei 4B (A); **n-max ist aufgabenabhängig** — kurz: mehr, lang: weniger, Code: deutlich höher (Community: n-max 11!) | `--spec-type draft-mtp --spec-draft-n-max N` |
| — MTP ist **verlustfrei** | Speculative Decoding verifiziert gegen das Basismodell — Ausgaben bitidentisch, Qualitätskosten: null | — |
| MoE-Experten auf CPU + Hot-Expert-Cache | A: 18→30 tok/s bei 35B (größter Hybrid-Hebel); B: bestätigt die Familie. **Achtung:** Cache-Größe ist nicht linear besser — siehe [Benchmarks](docs/benchmarks.md) | `--cpu-moe --moe-cache-mib N -lm none` |
| Build-Frische ist eine Stellschraube | gleiche Flags, neuer Build: 30→55 tok/s bei 43k (B, oneAPI-Sprung). Kernel-Updates kommen wöchentlich — lieber Repo-Release ziehen als Flags drehen | — |
| Kontext: 32k–48k ist bei modernen Architekturen fast gratis | 43k-Prompt: nur −10 bis −34 %; MTP-Acceptance steigt sogar mit Tiefe (beide Systeme) | `-c 49152` |
| Compute-Buffer statt Kontext opfern | Wenn der Kontext nicht passt: **`-ub/-b 1024` statt 2048** halbiert die Prefill-Buffer (Deckel: ~30 % Prefill-Tempo) und schafft so den Kontext — nicht der KV-Cache war der Engpass! | `-ub 1024 -b 1024` |

**Nicht pauschal versuchen:** `--moe-cache-cpu-share` ist maschinenabhängig (A langsamer,
B +80 %) — **CPU-Share misst man, man schätzt ihn nicht.** Ebenso nicht: `-fit`-Eingriffe
(läuft automatisch), Server-seitiges Reasoning-Abschalten, wenn der Client es per Request
steuern soll (s. u.).

## 6. An einen Coding-Agenten anbinden

Der Server spricht die OpenAI-API: `http://127.0.0.1:8901/v1`. Damit klappt fast jedes
Tool (Aider, Cline, Continue …). Beispiel opencode.jsonc (Ausschnitt):

```jsonc
{
  "providers": {
    "llamacpp-local": {
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": { "baseURL": "http://127.0.0.1:8901/v1" },
      "models": {
        "mein-modell": {
          "capabilities": { "tools": true, "reasoning": { "supported": true, "toggle": true, "effort": ["low","medium","high"] } },
          "limit": { "context": 49152, "output": 16384 },
          "variants": [
            { "id": "none", "settings": { "reasoningEffort": "none" } },
            { "id": "high", "settings": { "reasoningEffort": "high" } }
          ]
        }
      }
    }
  },
  "model": "llamacpp-local/mein-modell"
}
```

- **Thinking-Steuerung gehört dem Client/Agenten** (pro Request `reasoning_effort`) —
  nicht dem Server-Flag. llama.cpp versteht es nativ.
- **Anti-Loop-Regeln**: kleine Modelle loopen gern (73× derselbe Befehl in einer echten
  Session gezählt). Dagegen: `AGENTS.md` mit „gleicher Befehl max. 2×, dann Strategie
  wechseln", Variante `#none` für Bau-Aufgaben, kleine Sessions statt Marathon,
  `limit.output` hoch genug, damit Toolcalls nicht mittendrin abgeschnitten werden.
- **Compaction braucht Output-Raum:** Auto-Kompaktierung am Kontextdeckel stirbt, wenn
  keine Output-Tokens mehr frei sind („summary reached the output token limit“).
  Entweder früher manuell kompaktieren oder Server-Kontext größer als den Client-Limit
  fahren — der Client-Glaubenssatz steuert nur den Zeitpunkt, der Server-Luft die
  Qualität der Zusammenfassung.

### 6b. Mehrere Modelle ohne Neustart-Zirkus: llama-swap

Der klassische Schmerz: Modellwechsel = Server stoppen, Flags ändern, neu starten
(10 Minuten pro Wechsel). **[llama-swap](https://github.com/mostlygeek/llama-swap)**
löst das: ein Proxy, der pro Modellnamen einen llama-server mit **eigenen Flags**
startet/stoppt — Konfiguration komplett in einer `config.yaml`, Hot-Reload ohne Restart.

```yaml
# Ein Eintrag pro Modell = ein kompletter llama-server-Startbefehl
models:
  coder-moe:
    cmd: |
      llama-server --port ${PORT} -m /models/coder-35b.gguf
      -c 65536 -ngl 99 -fa on -ctk q8_0 -ctv q8_0
      --moe-cache-mib 4000 --spec-type draft-mtp --spec-draft-n-max 3
  quality-dense:
    cmd: |
      llama-server --port ${PORT} -m /models/quality-27b.gguf
      -c 131072 -ngl 99 -fa on -ctk q8_0 -ctv q8_0 -ub 1024 -b 1024
```

Client wählt per Modellnamen (`"model": "coder-moe"`) — der Swap lädt das Modell
on-demand, das vorherige wird entladen. **Erfahrungswerte:** Swap-Load eines
35B-MoE dauert 3–5 min (MoE-Cache-Aufbau + JIT) — die SSD für die Modelle drückt
das auf ~2 min. Erwartungsmanagement: nacheinander nutzbar, nicht parallel
(ein GPU-Modellprozess verdrängt den anderen; Koexistenz nur, wenn VRAM-Rest
reicht — llama-swap kann Gruppen, aber VRAM entscheidet).

Details und Produktions-Fallstricke: [docs/llama-swap.md](docs/llama-swap.md)

## 7. Fallstricke (Auswahl — vollständige Liste: [docs/troubleshooting.md](docs/troubleshooting.md))

1. **RAM-Druck ist der Killer**: vor Hybrid-Modellen RAM freiräumen, sonst streut die
   Geschwindigkeit chaotisch (A: 11–53 tok/s in identischer Config). Auch Zombies prüfen.
2. **Benchmarks sauber messen**: alten Server killen und *tot sein lassen* (bis 30 s
   Teardown), PID/Port/Flags verifizieren — sonst misst man Caches und Zombies.
3. **tok/s-Anzeige des Clients lügt**: Agenten zählen nur sichtbare Content-Tokens —
   Reasoning läuft unsichtbar mit. Wahrheit steht im Server-Log.
4. **Failed Loads hinterlassen Zombies mit vollem VRAM** — der nächste „Test auf leerer
   Karte“ läuft dann auf 5 % freiem Speicher und scheitert an ganz anderen Stellen.
   Nach jedem Failed-Load: Prozessliste prüfen.
5. **Host-IO-Stürme halten Modell-Loads an** (NAS/Server als Modell-Host): Parity-Checks
   und Mover-Jobs verhungern alles IO — erst Host-IO prüfen, dann am Modell doktern.
6. **Kein Hash, kein Modell**: GGUFs nach Download gegen die HF-Referenz-Hashes prüfen —
   „Dateigröße stimmt“ ≠ „Datei intakt“ (ein uncleaner reboot kann Datenblöcke verlieren,
   während die Größe stimmt). Und **keine Downloads parallel zum Modell-Load** auf
   derselben Platte — IO-Konkurrenz drückt den Load um das 6-fache.

## 8. Aussichten (was sich zu beobachten lohnt)

- **MTP/Ein-File-Draft-Heads** verbreiten sich über Modellfamilien — der größte
  Speed-Sprung der letzten 12 Monate auf Consumer-HW. Separate Draft-Dateien
  (Gemma-„assistant“-Drafter) folgen demselben Muster.
- **MoE mit kleinen aktiven Params** wird Standard: Cloud-Klasse-Qualität, hybrid
  auf 32 GB RAM. Die Grenze ist RAM-Bandbreite — DDR5/Quad-Channel profitiert doppelt.
- **KV-Innovationen** (Flash/DeltaNet) machen lange Kontexte auf 8-GB-Karten realistisch.
- **MoE-Cache ist mainline** und funktioniert außerhalb CUDA (SYCL ab oneAPI ≥ 2026.1
  bestätigt). „NVIDIA-only“-Annahmen alter Guides sind veraltet.
- **Agent-Integration**: mehr Clients lernen `reasoning_effort`-Toggles und
  AGENTS.md-Konventionen — lokale Agenten werden deutlich stabiler.
- **Hardware-Regel fürs nächste Upgrade**: erst RAM (Bandbreite/Volumen), dann VRAM,
  die CPU zuletzt — das war vor 2 Jahren genau andersrum.

---

## Danksagung

Ein Teil des Aufbaus und der Herangehensweise in diesem Repo basiert auf den Videos von
**[Kai Explains](https://www.youtube.com/@kaiexplainsYT)** (YouTube) — von dort stammen
entscheidende Denkanstöße, insbesondere rund um MoE-Caching und den lokalen
Coding-Agent-Stack. Dieses Repo ist unser Versuch, das dort Erklärte mit eigenen,
zweisystem-gemessenen Zahlen zu untermauern und auf das zu verdichten, was nach dem
Nachbauen wirklich übrig bleibt. Schaut bei ihm vorbei — die Videos erklären das *Warum*
oft zugänglicher als jede Tabellenzelle hier.

## Mitwirken

Messwerte, Korrekturen, Gegenbefunde — alles willkommen, siehe
[CONTRIBUTING.md](CONTRIBUTING.md). Besonders wertvoll: Messungen auf dritten
Systemen (AMD ROCm, Apple MLX, DDR5-Desktops), die die System-A/B-Faustregeln
testen oder falsifizieren.

## Lizenz

Inhalt: [CC BY 4.0](LICENSE) — teilen und bearbeiten mit Namensnennung erlaubt.

*Zusammengestellt aus zwei unabhängigen Messreihen. Zahlen: greedy-Messungen,
warm (nach JIT-Aufbau). Keine Cloud-Tokens wurden für dieses Dokument verbrannt,
alle Modelle laufen offline.*
