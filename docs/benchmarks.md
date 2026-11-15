# Benchmark-Rohdaten

Alle Messungen: **greedy, warm** (nach JIT/Cache-Aufbau), Cursor auf dem Server-Log
(`eval time … tokens per second`), nicht auf Client-UIs. Kontextangabe = belegter
Prompt-Kontext, nicht Fenstergröße.

## System B — KV-Cache-Quant-Matrix (35B-A3B MoE, 43k Kontext)

| KV-Typ | tok/s kurz | tok/s @43k | Bemerkung |
|---|---|---|---|
| q8_0 | 43,7 | 13–15,5 | DPAS/XMX-Kernel-Pfad |
| q5_1 | ~25 | 26,0 | „flach" über alle Tiefen + 2,8× Prefill |
| q5_0 | — | 6,7 | katastrophal — meidens |
| q4_0 | 14–23 | 12,4 | trotz „unterstützt"-Doku kaputt auf Hybrid-Arch (Head-Dims) |

**Muster:** der XMX/DPAS-FA-Kernel läuft nur auf q8_0/q4_0; alle anderen KV-Typen
fallen auf generische Kernel mit einem kontextunabhängigen Speed-Floor (~25 tok/s).
Doku-Angaben zur Kernel-Unterstützung sind kein Beweis — pro Modell/Backend messen.

## System B — Spekulative Decoding (MTP)

| n-max | kurz | @43k | Acceptance |
|---|---|---|---|
| — (off) | 34–48 | 13,0 | — |
| ngram (draftless) | — | 15,6 | 0,60 |
| draft-mtp n3 | 59–63 | 59,4 | 0,45–0,57 |

MTP ist verlustfrei (Outputs bitidentisch). Separate Draft-Dateien (kein Embedded-MTP)
sind mit `-md … --spec-draft-ngl all` äquivalent nutzbar.

## System B — MoE-Cache-Größe (Produktionsnachweis)

| moe-cache-mib | tok/s | Anmerkung |
|---|---|---|
| 4000 | ~60 stabil | bewiesener Produktionswert |
| 6000 | 36–46 | schlechter! Kontext: VRAM-Müll durch Failed-Loads in derselben Instanz — **nach MoE-Cache-Änderung immer warm nachmessen** |

Community-Daten (ROCm, 12 GB Karte, mit Draft): 1408→4288→5696 MiB = 41→67→74 tok/s.
Bigger is better *wenn* der VRAM-Room korrekt berechnet ist — bekanntes Bug-Muster:
mit geladenem Draft-Modell wird freier VRAM under-reportet (ggml issue #31911).

## System B — Speed-Matrix Kurzkontext (384-Tok-Runs, warm)

| Modell | tok/s | Thinking-Volumen | Anmerkung |
|---|---|---|---|
| 35B-A3B Coder MTP (MoE) | 68–76 | 4–8 KB | draft-mtp n3, moe-cache 4000 |
| 26B-A4B QAT (MoE, ohne MTP) | ~68 | 9–14 KB | A4B = nur 4B aktiv |
| 27B dense MTP | 24–26 | 0,3–4 KB | sparsames Thinking → oft zuerst fertig trotz halber Geschwindigkeit |

**End-to-End-Latenz folgt dem Thinking-Volumen, nicht den raw tok/s.**

## System B — Kontext-Skalierung (27B dense, Hybrid-Attention)

| Kontext | KV (q8_0) | Gesamt-VRAM | Status |
|---|---|---|---|
| 65k | ~1,9 GB | ~21,5 GB | komfortabel |
| 131k | ~3,7 GB | ~23,2 GB | lädt — nur mit `-ub/-b 1024` (Compute-Buffer waren der Engpass: 832 MiB zu viel bei ub 2048) |

Hybrid-Architekturen (Gated DeltaNet) haben nichtlineares KV-Wachstum — die
dichte-Klassiker-Faustregel „KV ≈ 0,5 MB/token" gilt hier nicht.

## System A — ausgewählte Messungen

| Maßnahme | Vorher | Nachher |
|---|---|---|
| `-lm none` (RAM-Load statt mmap), 35B hybrid | 18 tok/s | 30 tok/s (+67 %) |
| MTP n-max 2, 9B | 55 tok/s | 73 tok/s (+34 %) |
| MTP, 4B | — | 112 tok/s |
| CPU-MoE + Hot-Expert-Cache, 35B | 18 tok/s | 30 tok/s |

## Load-Zeiten (System B, HDD-Modellspeicher, ohne IO-Konkurrenz)

| Modell | Load | Bemerkung |
|---|---|---|
| 35B-A3B (MoE-Cache + MTP) | 4–5 min | MoE-Cache-Aufbau + JIT dominiert |
| 27B dense | 4–5 min | |
| 26B-A4B | ~3 min | |

Mit NVMe statt HDD: ~2 min (erwartet, unbestätigt). **Keine Downloads/Hash-Jobs
parallel zum Modell-Load** — IO-Konkurrenz drückt den Load von 243 MB/s auf ~40 MB/s.
