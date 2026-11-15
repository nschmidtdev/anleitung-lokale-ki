# ANLEITUNG — das agent-ausführbare Playbook

Diese Datei ist für KI-Agenten (OpenCode, Claude Code, Hermes, …) geschrieben: Phasen
mit **exakten Befehlen**, dem **erwarteten Ergebnis** und dem **Verzweigungsverhalten**,
wenn etwas abweicht. Arbeite die Phasen der Reihe nach ab; überspringe nichts und
verifiziere jeden Gate, bevor du weitermachst.

**Konventionen in dieser Datei:**
- `$VDIR` = dein Modell-Verzeichnis (z. B. `/models` oder `D:\models`), setze es einmal
- `$PORT` = Server-Port (Standard: 8901)
- „Gate ✓" = gemessenes/erwartetes Ergebnis. Weicht es ab: erst dem Abzweig folgen,
  nie weitermachen mit „geht schon“.
- Regel-Referenz für Konzepte: [README.md](README.md). Symptom-Suche:
  [docs/troubleshooting.md](docs/troubleshooting.md).

---

## Phase 0 — Hardware-Check (Gate: genug VRAM/RAM bekannt)

```bash
# NVIDIA:
nvidia-smi --query-gpu=memory.total --format=csv
# Intel Arc (Linux):
clinfo | grep -i "global memory" ; ls /dev/dri/
# RAM:
free -h          # Linux
wmic OS get TotalVisibleMemorySize   # Windows
```

**Entscheidung** (VRAM → Modell-Klasse, Details README Kap. 4):

| VRAM frei | Modell-Klasse | quantisiertes GGUF ≈ |
|---|---|---|
| ≥ 7,5 GB | 4–10B dense oder ~3B-aktiv-MoE | ≤ 6,5 GB |
| ≥ 14 GB | 14–32B dense, 30B-A3B | ≤ 13 GB |
| ≥ 22 GB | 32B dense Q4, 35B-A3B komplett | ≤ 19 GB |
| < GPU, viel RAM | Hybrid mit `--n-cpu-moe` / MoE-Cache | beliebig, RAM-Bandbreite ist das Limit |

**Abzweig:** RAM < Dateigröße + 4 GB → wähle kleineres Modell. Kein GPU-Modell-Trade
ohne Bandbreiten-Check (`dmidecode -t memory | grep Speed` — unter 2400 MT/s:
kleines Modell komplett auf GPU schlägt großes hybrid).

### Pflicht-Gate: den Nutzer befragen, BEVOR ein Modell gewählt wird

Ein Agent soll hier **nicht selbst entscheiden** — Modellwahl ist eine
Präferenz-Entscheidung mit Trade-offs. Frage den Nutzer (alle Fragen in einem
Durchgang, mit den gemessenen Konsequenzen):

1. **Passungsverhalten:** „Modell komplett in die Grafikkarte (schnell, aber
   Größen-Limit) oder großes Modell mit Auslagerung auf CPU/RAM (mehr Qualität,
   langsamer — Faktor 2–3)?" → entscheidet Quant/Größe und ob `--n-cpu-moe`/
   MoE-Cache/Strata-Themen relevant werden
2. **Priorität:** „Speed (Agent-Arbeit, flüssiges Arbeiten) oder Qualität (seltenere,
   schwerere Aufgaben)?" → klein-schnell vs. groß-langsam
3. **Kontextbedarf:** „Wie lang sind typische Sessions/Prompts? (Chat < 8k,
   Coding-Agent 32–65k, Riesen-Kontext > 100k)" → entscheidet `-c` und ob
   Hybrid-Architekturen (DeltaNet/sliding window) Pflicht sind
4. **Inhalt:** „Allround, Coding-Finetune oder uncensored?" → Finetune-Klasse
5. **Sonderfälle:** Vision? FIM/Inline-Completion (dann BASE-Modell, nie Instruct)?
   Mehrere Modelle (→ llama-swap, Phase 5)?

Mappe die Antworten auf VRAM-Tabelle + Quant-Sweet-Spot (README Kap. 3/4) und nenne
dem Nutzer 1–2 konkrete Kandidaten mit den Trade-offs — **dann** weiter zu Phase 1/2.
Erst nach seiner Wahl herunterladen und konfigurieren.

---

## Phase 1 — Engine installieren (Gate: `llama-server --version` läuft)

**Die Engine hängt an der GPU-Hersteller — die richtige Variante nehmen:**

| GPU darunter | llama.cpp-Image (ggml-org) | llama-swap-Image (mostlygeek) | Anmerkung |
|---|---|---|---|
| NVIDIA | `ghcr.io/ggml-org/llama.cpp:server-cuda` | `ghcr.io/mostlygeek/llama-swap:cuda` | der gepflasterte Weg |
| **Intel Arc/Battlemage** | `ghcr.io/ggml-org/llama.cpp:server-intel` | `ghcr.io/mostlygeek/llama-swap:intel` | SYCL/oneAPI **im Image** (≥ 2026.1!), kein Host-oneAPI nötig; GPU-Devices `/dev/dri` durchreichen |
| AMD (ROCm) | `ghcr.io/ggml-org/llama.cpp:server-rocm` | `ghcr.io/mostlygeek/llama-swap:rocm` | HSA_OVERRIDE_GFX_VERSION je nach Karte |
| AMD alt/sonstige, CPU-only | `:server-vulkan` / `:server` | `:vulkan` / `:cpu` | Vulkan meist die schnellste Notlösung |
| Apple Silicon | nativ (`brew install llama.cpp`) bzw. MLX | — | MTP-Heads teils nur dort |

**Niemand mischt:** ein CUDA-Image auf einem Arc-Host findet keine GPU („no known GPU
vendor"), und CA-Templates tragen oft NVIDIA-Leftovers (`--gpus all`) — bei Intel/AMD
strippen und stattdessen `/dev/dri`-Devices durchreichen.

```bash
docker pull ghcr.io/ggml-org/llama.cpp:<VARIANTE-AUS-TABELLE>
```

**Weg B — Native:** Release-Tarball von
`github.com/ggml-org/llama.cpp/releases` (zum Backend passende Variante).

**Gate ✓:** `llama-server --version` gibt eine Build-Nummer (b####).
**Abzweig:** Version älter als ~4 Wochen → erst Repo-Update ziehen. Build-Frische
ist eine gemessene Stellschraube (bis +80 % Decode), bevor du Flags drehst, aktualisiere.

---

## Phase 2 — Modell laden & verifizieren (Gate: Hash matcht)

```bash
# 1. Dateigröße vorab via HF-API holen (GB vermeiden — byte-exakt):
curl -s "https://huggingface.co/api/models/<REPO>/tree/main" | grep <DATEINAME>.gguf

# 2. Download:
curl -L -o $VDIR/<DATEINAME>.gguf "https://huggingface.co/<REPO>/resolve/main/<DATEINAME>.gguf"

# 3. Verifizieren — beides, immer:
du -b $VDIR/<DATEINAME>.gguf                 # == API-Größe?
head -c 4 $VDIR/<DATEINAME>.gguf | od -c     # muss "G G U F" zeigen
sha256sum $VDIR/<DATEINAME>.gguf             # gegen HF/README-Referenz
```

**Gate ✓:** Größe byte-exakt, Magic `GGUF`, Hash matcht.
**Abzweige:**
- Größe stimmt, Hash nicht → Datei löschen, neu laden. Nie „geht schon“ bei Models >10 GB.
- Modell soll Spec-Decoding (MTP) nutzen → Header prüfen:
  `head -c 64M <DATEI>.gguf | tr -c "[:print:]" "\n" | grep -aE "nextn"` — kein Treffer
  = kein Embedded-MTP, `--spec-type draft-mtp` wird fehlschlagen.

---

## Phase 3 — Erster Start mit sicheren Flags (Gate: 200 auf /health)

```bash
llama-server -m $VDIR/<MODEL>.gguf --port $PORT --host 127.0.0.1 \
  -c 65536 -np 1 -ngl 99 -fa on -ctk q8_0 -ctv q8_0 \
  -t 8 -tb 16 --api-key <KEY>
```

**Gate ✓ (in dieser Reihenfolge prüfen):**
```bash
curl -s http://127.0.0.1:$PORT/health                                        # → 200
curl -s http://127.0.0.1:$PORT/v1/chat/completions -H "Content-Type: application/json" \
  -H "Authorization: Bearer <KEY>" \
  -d '{"messages":[{"role":"user","content":"Sag OK"}],"max_tokens":4}'      # → echtes JSON
docker top <CONTAINER> -o pid,args | grep llama-server                        # Flags = ground truth
```

**Abzweige:**
- OOM beim Load → `[SYCL-MEMTRACE] alloc FAILED` lesen: steht dort ein Betrag ~1 GB
  mit hoher „already allocated“-Summe → **Compute-Buffer halbieren** (`-ub 1024 -b 1024`),
  NICHT den Kontext schrumpfen. KV selbst ist selten der Engpass.
- `tensor data is not within the file bounds` → Datei korrupt (Phase 2 wiederholen;
  uncleaner Reboot kann Größe-richtig-Inhalt-kaputt hinterlassen).
- Load hängt bei 0 % CPU → Host-IO-Sturm prüfen (NAS-Parity, Mover, Backups) —
  erst IO freimachen, dann neu starten.

---

## Phase 4 — Tuning in fester Reihenfolge (Gate: je Schritt warm messen)

Messbefehl (immer warm = zweiter Request nach Load; greedy; Server-Log lesen):

```bash
curl -s http://127.0.0.1:$PORT/v1/chat/completions -H "Content-Type: application/json" \
  -H "Authorization: Bearer <KEY>" \
  -d '{"messages":[{"role":"user","content":"<≥1500 Token Prompt>"}],"max_tokens":384,"stream":false}' \
  | grep -o '"per_second":[0-9.]*'
```

Reihenfolge (nur ein Flag-Zug pro Messung, Ergebnis gegen Baseline vergleichen):

1. `-fa on` (Baseline haben) → 2. KV `q8_0` → 3. MTP falls im Header
   (`--spec-type draft-mtp --spec-draft-n-max 2`, sweep 2/3/6 — kurz/lange/Code-Aufgaben
   haben verschiedene Optima) → 4. bei RAM-Spill: `--moe-cache-mib N` (N = Spill-Größe ±,
   **immer warm nachmessen — größer ist nicht besser**) → 5. `-ub/-b` hoch bis OOM-Grenze.

**Regeln:**
- KV-Typ pro Backend/Modell messen — q5_0 kann katastrophal sein, q5_1 „flach“,
  q8_0 hat oft den Schnell-Kernel. Doku-Angaben sind kein Beweis.
- `--moe-cache-cpu-share` nur mit Messung setzen (maschinenabhängig, 0 vs 0,9 =
  jeweils große Unterschiede in beide Richtungen).
- **Keine Downloads/Hash-Jobs/Backups parallel zum Load** — erst beenden (scp, rsync),
  dann laden. IO-Konkurrenz drückt Loads um das 6-fache.
- Failed Load → Zombie-Check (Prozessliste/`docker restart`), sonst läuft der
  nächste „Leer-Test“ auf 5 % VRAM und scheitert woanders.

---

## Phase 5 — Multi-Modell mit llama-swap (Gate: Modellwechsel per Name)

```bash
# config.yaml: ein Eintrag pro Modell, kompletter Startbefehl (siehe docs/models.md)
# Container/Proxy starten:
llama-swap -config config.yaml -watch-config
```

**Gate ✓:**
```bash
curl -s http://127.0.0.1:$PORT/v1/models          # alle Einträge gelistet
# Request auf Modell B entlädt A automatisch — Load dauert 3-5 min (MoE: normal!)
```

**Regeln:** `healthCheckTimeout ≥ 900`. Failed-Spawn-Fehler lesen: „prematurely“ =
GGUF/OOM (Phase-3-Abzweige); „failed to create MTP context“ = draft-mtp-Flags aus
diesem Eintrag entfernen. Config-Änderung braucht nur Model-Unload, keinen Restart.

---

## Phase 6 — Client anbinden (Gate: Agent-Aufgabe end-to-end)

Client auf `http://127.0.0.1:$PORT/v1` zeigen (Beispiel-Konfig: README Kap. 6).

**Gate ✓:** eine kleine Agent-Aufgabe (Datei lesen + editieren + Test ausführen)
läuft ohne Tool-Call-Fehler durch.

**Abzweig:** Toolcalls scheitern oder Modell „simuliert“ Calls → Chat-Template und
Parser-Format abgleichen (`--jinja`, JSON-native vs. XML) — falsches Alignment macht
fähige Modelle unbrauchbar. Dann erst Modell-Kritik.

---

## Symptom → Check (schnelle Nachschlage)

| Symptom | Erster Check | Dann |
|---|---|---|
| „lädt nicht, Karte leer“ | IO-Sturm auf dem Modell-Host? | mmap-Phase liest Platte — RSS wächst? Dann normal, warten |
| plötzlich halbe tok/s | alte Server-Prozesse tot? RAM frei? | Zombies killen, 30 s Teardown abwarten, neu messen |
| Antwort bleibt leer, `finish=length` | max_tokens zu klein fürs Thinking | ≥4000 setzen, `content`-Kanal lesen |
| Compaction/Summary bricht ab | Output-Raum am Kontextdeckel | Server `-c` größer als Client-Limit (Phase 5, `-ub 1024`) |
| 500 `reasoning effort high` | Template kennt Stufe nicht | Client auf medium/low (Qwen: xhigh/medium/low) |

Vollständig: [docs/troubleshooting.md](docs/troubleshooting.md) ·
Mess-Basis: [docs/benchmarks.md](docs/benchmarks.md) ·
Produktions-Beispiel: [docs/models.md](docs/models.md)
