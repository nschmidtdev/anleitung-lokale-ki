# AGENTS.md — Anweisungen für KI-Agenten in diesem Repo

**Wenn du (als Agent) hier arbeitest oder der Nutzer Hilfe beim Aufbau braucht:**

1. **Der ausführbare Playbook ist [`ANLEITUNG.md`](ANLEITUNG.md)** — Phasen 0–6
   mit exakten Befehlen, Gates und Verzweigungen. Bei Setup-/Optimierungs-Aufgaben:
   Phase der Reihe nach, jeden Gate verifizieren, nie „geht schon“ weitermachen.
2. Mess-Daten und Herleitungen: [`docs/benchmarks.md`](docs/benchmarks.md) —
   Claims hier sollen mit Zahlen belegt sein, nicht mit Meinungen.
3. Symptom-Suche vor eigenem Debugging: [`docs/troubleshooting.md`](docs/troubleshooting.md).
4. Produktionsbeispiel (Modelle + komplette Config): [`docs/models.md`](docs/models.md).

**Konventionen für Contributions:**
- Messwerte mit Hardware/Build-Angabe, warm gemessen, Server-Log als Quelle —
  Details in [CONTRIBUTING.md](CONTRIBUTING.md).
- Keine Secrets/Keys/IPs in Logs, Configs oder Commits.
- Deutsch/Englisch beide ok; Dokumente auf Deutsch, Code-Kommentare beliebig.
