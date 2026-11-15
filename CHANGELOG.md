# Changelog

## v1.1 — November 2026

- `docs/models.md`: bewährtes Modell-Trio dokumentiert — Rollen (Speed-Coder /
  Präzision / Balance), Hugging-Face-Links zu den exakten GGUF-Repos, gemessene
  tok/s, Thinking-Profile und die komplette llama-swap-Config als Copy-Paste-Basis
- Caveats ergänzt: Gemma-26B-QAT ohne draft-MTP fahren, 131k-Kontext braucht
  `-ub/-b 1024` (Compute-Buffer statt KV war der Engpass), Hash-Check-Pflicht

## v1.0 — November 2026

- Erste Veröffentlichung: Leitfaden aus zwei unabhängigen Messreihen
  (Consumer-RTX-2080-System + 24-GB-Intel-SYCL-Server)
- Kapitel: Wirtschaftlichkeit, Hardware, Software-Wege, Modell-Größenregeln,
  getunte Stellschrauben (KV-Quant, MTP, MoE-Cache, Build-Frische, Kontext),
  Coding-Agent-Anbindung, llama-swap-Multi-Modell-Betrieb, Fallstricke, Aussichten
- Benchmark-Rohdaten: KV-Quant-Matrix, MTP-Sweeps, MoE-Cache-Größen,
  Kurzkontext-Matrix, Kontext-Skalierung, Load-Zeiten
