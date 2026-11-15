# Unser bewährtes Modell-Trio (Stand: November 2026)

Drei Modelle, drei Rollen, ein llama-swap-Proxy — alle Messwerte aus der
produktiven Nutzung (24-GB-GPU, Intel SYCL, Details: [benchmarks.md](benchmarks.md)).
Links führen zu den exakten GGUF-Repos.

## Die Rollen

| Rolle | Modell | Quant | Größe | tok/s (kurz) | Thinking | Kontext |
|---|---|---|---|---|---|---|
| **Speed-Coder** (Agent-Arbeit) | [Qwopus3.6-35B-A3B „APEX-Mini"](https://huggingface.co/LuffyTheFox/Qwopus3.6-35B-A3B-Coder-Uncensored-Genesis-MTP-GGUF) | Preservation-Mix (Q6_K/Q5_K/F32) | 20,7 GB | **68–76** (59 bei 43k) | mittel (4–8 KB) | 65k (VRAM-Grenze) |
| **Präzision** (kurze, genaue Aufgaben) | [Qwen3.8-27B „HauhauCS-Aggressive“](https://huggingface.co/HauhauCS/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-MTP-GGUF) | Q4_K_P | 17,9 GB | 24–26 | **sparsam** (0,3–4 KB) | 131k (hybrid-KV wächst langsam) |
| **Balance** (allg. Chat/Code) | [Gemma4-26B-A4B „QAT-Balanced“](https://huggingface.co/HauhauCS/Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-MTP) | QAT Q4_K_M | 16,8 GB | ~68 | **Grübler** (9–14 KB) | 131k+ (sliding window) |

Warum diese drei? Der Coder liefert Agent-Geschwindigkeit, der Qwen die beste
Regel-Adhärenz mit minimalem Thinking-Overhead (im Benchmark 43/51 gegen Claude
Sonnet 5), die Gemma den besten Kompromiss aus Qualität und Tempo bei allgemeinen
Aufgaben. Alle drei: uncensored-Finetunes der Qwen3.6/3.8- bzw. Gemma4-Familie.

**Wichtige Caveats:**
- Die 26B-QAT-Gemma akzeptiert **kein draft-MTP** (`failed to create MTP context`) —
  ohne die Flags fahren, kostet ~25 % Speed. Wer Gemma-MTP will: die 31B-Varianten
  bringen einen funktionierenden separaten Draft (`mtp-gemma-4-31B-it.gguf`).
- Q4_K_P ist ein „Aggressive“-Maßnahme-Quant (gemessene Allokation, kein Standard-Schema).
- Downloads **immer gegen die HF-Referenz-Hashes prüfen** — siehe troubleshooting.md.

## Die llama-swap-Config dazu (Copy-Paste-Basis)

```yaml
healthCheckTimeout: 900
startPort: 5800

models:
  qwopus-coder:            # Speed-Coder, 65k = VRAM-Grenze (20,7 GB + 4 GB Cache)
    cmd: |
      llama-server --host 127.0.0.1 --port ${PORT}
      -m /models/Qwopus3.6-35B-A3B-Uncensored-Genesis-Hermes-APEX-Mini.gguf
      -c 65536 -np 1 -ngl 99 -fa on -ctk q8_0 -ctv q8_0 -t 8 -tb 16
      -ub 2048 -b 2048
      --moe-cache-mib 4000 --moe-cache-cpu-share 0.9
      --spec-type draft-mtp --spec-draft-n-max 3
      --api-key <API-KEY>
    ttl: 0

  qwen38-27b-mtp:          # Präzision, 131k (Compute-Buffer halbiert, sonst OOM)
    cmd: |
      llama-server --host 127.0.0.1 --port ${PORT}
      -m /models/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-Q4_K_P.gguf
      -c 131072 -np 1 -ngl 99 -fa on -ctk q8_0 -ctv q8_0 -t 8 -tb 16
      -ub 1024 -b 1024
      --spec-type draft-mtp --spec-draft-n-max 2
      --api-key <API-KEY>
    ttl: 0

  gemma4-a4b-qat:          # Balance — OHNE draft-mtp (siehe Caveat oben)
    cmd: |
      llama-server --host 127.0.0.1 --port ${PORT}
      -m /models/Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-Q4_K_M.gguf
      -c 131072 -np 1 -ngl 99 -fa on -ctk q8_0 -ctv q8_0 -t 8 -tb 16
      -ub 1024 -b 1024
      --api-key <API-KEY>
    ttl: 0
```

Anpassen an deine Maschine: Thread-Zahlen (`-t/-tb`), Kontext nach VRAM-Budget,
MoE-Cache nach Spill-Größe (siehe [benchmarks.md](benchmarks.md), „immer warm
nachmessen“).

## Was das Trio gekostet hat (ehrliche Rechnung)

- **Swaps sind Wartezeit**: jeder Modellwechsel = 3–5 min Load (MoE-Cache + JIT).
  Mit NVMe ~2 min. Die Modelle nacheinander statt ständig wechseln.
- **Rule-Adhärenz ist kein Unterscheidungskriterium**: alle drei schafften 9/9 in
  einem strikten Format-/Code-/Satz-Benchmark. Der echte Unterschied liegt im
  Thinking-Volumen (→ End-to-End-Latenz) und im Code-Stil.
- **Compaction-Raum einplanen**: der Client-Limit steuert nur den Kompaktierzeitpunkt;
  dem Server genug Output-Raum lassen (Details: [llama-swap.md](llama-swap.md),
  „Kontextstrategie“).
