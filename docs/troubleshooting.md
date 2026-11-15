# Troubleshooting — Fallstricke aus dem echten Betrieb

Gesammelte Stolpersteine, sortiert nach „wie oft hat's uns gebissen“.

## Server & Prozesse

1. **RAM-Druck ist der Killer.** Vor Hybrid-Modellen RAM freiräumen, sonst streut die
   Geschwindigkeit chaotisch (gemessen: 11–53 tok/s in identischer Config). Auch
   Zombie-Prozesse prüfen: alte Server halten RAM/VRAM.
2. **Benchmarks sauber messen.** Vor jeder Messung alten Server killen und *tot sein
   lassen* — große Modelle brauchen bis 30 s Teardown. PID/Port/Flags verifizieren,
   sonst misst man Caches und Zombies, nicht das Modell.
3. **Failed Loads hinterlassen Zombies mit vollem VRAM.** Der nächste „Test auf leerer
   Karte“ läuft auf 5 % freiem Speicher und scheitert an ganz anderen Stellen — die
   Fehlermeldung zeigt dann Symptome, nicht Ursachen. Nach jedem Failed-Load:
   Prozessliste prüfen bzw. den Container/Proxy neu starten.
4. **VRAM-OOM richtig lesen.** `[SYCL-MEMTRACE] alloc FAILED: … X MiB already
   allocated, Z MiB usable` verrät, *was* fehlt: Nicht immer ist der KV-Cache der
   Engpass — oft sind es die Compute-Prefill-Buffer (bei `-ub/-b 2048`). Halbiert man
   die (`-ub/-b 1024`), passt oft deutlich mehr Kontext, ohne Geschwindigkeit beim
   Decode zu verlieren.
5. **API-Keys aus Prozesslisten fernhalten.** `docker top`/`ps` zeigen Kommandozeilen
   inklusive `--api-key` — Secrets über Env-Variablen oder Proxy-Config fahren, wenn
   möglich.

## Messen & Interpretieren

6. **tok/s-Anzeige des Clients lügt.** Agenten zählen nur sichtbare Content-Tokens —
   Reasoning-Tokens laufen unsichtbar mit voller Server-Geschwindigkeit mit. Die Wahrheit
   steht im Server-Log (`prompt eval / eval time`).
7. **Reasoning-Modelle brauchen große max_tokens.** Ein 2000er-Budget lässt das Thinking
   die ganze Antwort fressen: `finish_reason=length`, `content` leer — sieht aus wie
   Regelverstoß/QA-Ausfall, ist Budget-Mord. Vor dem Scoren `finish_reason` prüfen,
   Antwort liegt auf dem `content`-Kanal, Reasoning nur zur Diagnose lesen.
8. **Rule-Checks gegen alternative Schreibweisen auslegen.** „6k±1-Optimierung“ kommt
   als `i += 6` mit `i` und `i+2` daher, nicht als Literal „6k“. Automatische Benchmarks
   sonst falsch negativ.

## Downloads & Storage

9. **Kein Hash, kein Modell.** GGUFs nach Download gegen HF-Referenz-Hashes prüfen
   (SHA256). „Dateigröße stimmt“ ≠ „Datei intakt“: ein uncleaner Reboot kann Datenblöcke
   verlieren, während die Größe stimmt — llama.cpp meldet dann `tensor data is not
   within the file bounds`.
10. **Keine Downloads parallel zum Modell-Load** auf derselben Platte. IO-Konkurrenz
    drückt den Modell-Load um das 6-fache (243 → 40 MB/s gemessen), bis der Load-Proxy
    in sein Timeout läuft. Erst Hintergrund-Transfers stoppen, dann laden lassen.
11. **Hugging-Face-Downloads verifizieren**: Dateigröße vorab via API abfragen, danach
    byte-exakt vergleichen (ein falscher Repo-Pfad liefert HTTP 200 mit einem 29-Byte-
    Fehlerkörper als `.gguf`).
12. **FIM/Inline-Completion braucht BASE-Modelle, nie Instruct** — Instruct-Varianten
    geben am FIM-Endpoint Chat-Prosa statt Code aus.

## Host-Umgebung

13. **Host-IO-Stürme halten Modell-Loads bei 0 % CPU an** (wenn der Modell-Host ein
    Server/NAS ist): Parity-Checks, Recovery-Jobs, Mover-Läufe verhungern alle IO.
    Erst Host-IO prüfen, dann am Modell herumdoktern — das Symptom sieht aus wie
    „das Modell lädt nicht“, ist aber Storage.
14. **Unclean Reboot = Hash-Check für alles Frisch geschriebene.** Große Dateien, die
    kurz vor einem Absturz/Stop geschrieben wurden, können Größe-richtig, Inhalt-kaputt
    sein (Write-Back-Cache wurde nicht geflusht).
15. **Windows-Quirks aus dem Terminal**: `taskkill` aus Git-Bash schlägt still fehl
    (Python-Wrapper nutzen), native Tools verstehen keine MSYS-Pfade (`D:/...` statt
    `/d/...`), deutsche Konsolenausgabe ist CP850.

## Client-Integration

16. **Compaction braucht Output-Raum.** Auto-Kompaktierung am Kontextdeckel stirbt mit
    „summary reached the output token limit“, wenn kein Output-Token mehr frei ist.
    Server-Kontext größer als Client-Limit fahren oder früher kompaktieren.
17. **Tool-Call-Template-Alignment ist alles.** Ein Agent-Test mit falschem
    Chat-Template/Parser macht ein fähiges Modell „hoffnungslos“ (malfunctioning tool
    calls, Runaways). Bevor man dem Modell die Schuld gibt: Template und Parser-Format
    (JSON-native vs. XML) abgleichen — nach einem Fix war dasselbe Modell ein 12/12-
    Perfektionist.
18. **Modellnamen im Client sind meist kosmetisch** — geladen ist das GGUF des
    laufenden Servers. Aliase/`--alias` lösen die Anzeige.
