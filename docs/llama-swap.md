# llama-swap in Produktion — Erfahrungen und Fallstricke

[llama-swap](https://github.com/mostlygeek/llama-swap) ist ein Proxy, der pro
Modellnamen einen llama-server mit eigenen Flags startet/stoppt. Nach der Migration
von „ein Container pro Modell“ ist es die empfohlene Architektur für Multi-Modell-Setups
auf einer Karte.

## Warum es sich lohnt

- Modellwechsel = Modellnamen ändern, **kein Container-Neustart**
- Ein Eintrag pro Modell mit komplettem Startbefehl — jedes Modell seine eigenen
  Flags (Kontext, KV-Quant, Spec-Decoding, MoE-Cache)
- `-watch-config`: Änderungen an der config.yaml werden live übernommen; sie greifen
  beim nächsten Load des Modells. Ein **Container-Restart ist nur bei Image-Updates
  oder VRAM-Müll nötig** — kein Muss für Parameter-Änderungen
- `aliases:` halten alte Client-Modellnamen am Leben (null-touch Migration)

## Setup-Grundlagen (funktionierendes Muster)

```yaml
healthCheckTimeout: 900   # MoE auf SYCL lädt 3-5 MIN — Default-Timeout killt die Loads
logLevel: info
startPort: 5800           # interner Port-Pool für die Spawn-Prozesse

models:
  coder-moe:
    cmd: |
      llama-server --host 127.0.0.1 --port ${PORT}
      -m /models/coder-35b.gguf -c 65536 -ngl 99 -fa on
      -ctk q8_0 -ctv q8_0 -t 8 -tb 16 -ub 2048 -b 2048
      --moe-cache-mib 4000 --spec-type draft-mtp --spec-draft-n-max 3
    aliases: [alter-name-vom-legacy-server]
    ttl: 0                # 0 = nie per Idle-Timeout entladen
```

Docker-Seite: nur Infrastruktur im Template/Container — Image, Port, Mounts
(`models: ro`, `config: rw`), GPU-Devices (`/dev/dri`). **Kein** KI-Kram mehr im
Container — kein `-m`, keine Flags, kein Modellname.

## Produktions-Fallstricke (alle selbst erlebt)

1. **`starting X failed: upstream command exited prematurely`** = der Spawn ist
   abgesoffen. Häufigste Ursachen: GGUF defekt (`tensor data is not within the file
   bounds` → Hash prüfen) oder OOM im Spawn (siehe troubleshooting.md, VRAM-OOM lesen).
2. **`failed to create MTP context`** = GGUF trägt keine verwertbaren MTP-Tensoren für
   diese Architektur (z. B. manche QAT-Varianten). draft-mtp-Flags aus dem cmd-Block
   entfernen — das Modell läuft dann normal, nur ohne Spec-Decoding.
3. **`health check timed out` während IO-Sturm** = der Load war zu langsam, nicht kaputt.
   Wenn der IO-Sturm vorbei ist, einfach erneut anfragen — der Load startet neu.
4. **Config-Änderung + Model-Unload statt Restart:** llama-swap killt bei Config-Änderung
   das laufende Modell, der nächste Request spawnt mit den neuen Flags. Container-Restart
   nur bei Image-Update oder wenn nach fehlgeschlagenen Loads VRAM-Müll hängen bleibt.
5. **„No GPU data available“ im UI ist bei Intel/AMD normal** — das GPU-Monitoring liest
   nvidia-smi; die Modelle laufen trotzdem auf der GPU (an den tok/s erkennen).
6. **Load-Phasen-Anatomie:** bei MoE-Modellen bleibt die GPU bis zum Schluss leer —
   erst wird die ganze GGUF von Platte gelesen (mmap-Phase, sichtbar als wachsendes
   RSS des Spawn-Prozesses), dann geht der VRAM-Upload in Sekunden. „Karte bleibt
   leer“ beim Laden ist kein Fehler.
7. **Health-Check-Timeout großzügig setzen** (900 s): MoE-Cache-Aufbau + SYCL-JIT
   dauern; das Default-Timeout killt regelmäßig gesunde Loads.
8. **OpenAI-Clients können bei Custom-Providern NICHT `/v1/models` auto-discovern**
   (OpenCode openai-compatible z. B.) — Modellliste explizit in der Client-Config
   pflegen, `limit.context` dort real setzen.
9. **Nacheinander statt parallel:** Ein GPU-Modellprozess verdrängt standardmäßig den
   anderen. Gruppen (`swap: false` für Koexistenz) nur nutzen, wenn das VRAM-Budget
   der Summe aller Mitglieder passt — llama-swap rechnet kein VRAM aus.

## Kontextstrategie: Client-Glaube vs. Server-Realität

Der im Client konfigurierte `limit.context` steuert nur, wann der Client kompaktiert.
Der **Server** kann ein größeres Fenster haben — dann bekommt die Compaction-Zusammenfassung
genug Output-Raum, obwohl der Client „am Limit“ glaubt. Getestetes Muster:

- Client: `limit.context: 65536` (kompaktiert früher, gutmütig)
- Server: `-c 131072` mit `-ub/-b 1024` (Compute-Buffer statt Kontext opfern — der
  KV-Cache war nie der Engpass)

Ergebnis: Compaction-Fehler („summary reached the output token limit“) verschwinden,
ohne den Client anzufassen.
