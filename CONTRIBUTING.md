Mitmachen ist ausdrücklich erwünscht. Dieses Repo lebt von dritten Messreihen.

## Was besonders wertvoll ist

1. **Messungen auf Systemen, die hier fehlen**: AMD ROCm, Apple MLX/Unified, DDR5-
   Desktops, Quad-Channel-DDR4 — immer dort, wo die System-A/B-Faustregeln (z. B.
   „RAM-Bandbreite entscheidet Hybrid-Tempo“, „CPU-Share misst man, man schätzt ihn
   nicht“) bestätigt oder **falsifiziert** werden. Falsifikationen sind Gold wert.
2. **KV-Quant-Matrizen pro Backend**: die Kernel-Pfade (DPAS vs. generisch) sind
   backend- und modellabhängig — eine Tabelle für ROCm/Vulkan/Metal würde die
   bestehende ergänzen.
3. **Draft-n-max-Sweeps** pro Modellfamilie (kurze Antworten vs. lange vs. Code) —
   die Optima sind wild unterschiedlich (gemessen: n-max 1 bis 11 je nach Modell/Task).
4. Korrekturen: alles mit Zahlen, bitte — „ging bei mir schneller“ ohne Messaufbau
   hilft niemandem.

## Format für Mess-PRs

- Hardware kurz (GPU, RAM-Typ/-Bandbreite, Plattform/Backend)
- llama.cpp-Build/Version (die ändert Kernel-Verhalten!)
- Messaufbau: Prompt-Größe, Output-Tokens, warm (JIT/Ladezeit raus), greedy oder
  Sampling-Parameter, Messquelle (Server-Log, nicht Client-UI)
- 2–3 Wiederholungen oder Hinweis auf Single-Run

## Regeln

- Deutsch oder Englisch, beides ok (gerne issue auf Englisch stellen, wir übersetzen).
- Keine Modellnamen-Diskussionen ohne Messbasis — Namen altern, Größenregeln nicht.
- Keine API-Keys, IP-Adressen oder Hostnamen in Logs/Configs, die eingereicht werden.
