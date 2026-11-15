# Alternative: GPU mieten statt kaufen (vast.ai & Co.)

Eine 24-GB-Klasse-GPU muss nicht gekauft werden — auf GPU-Marktplätzen wie
[vast.ai](https://vast.ai/pricing) mietest du Karten stundenweise (Preis je GPU-Stunde +
Storage + Traffic; der Client zahlt ~33 % über dem Host-Preis, Stand 11/2026). Diese
Seite sortiert, wann das lohnt — und wann nicht.

## Preise (November 2026, Trackerschnitt aus 3 Quellen)

| Karte | VRAM | typisch $/h | Floor (Spot/Community) | Für unser Thema |
|---|---|---|---|---|
| RTX 3090 | 24 GB | $0,15–0,25 | $0,13 | 24-GB-Klasse billig |
| RTX 4090 | 24 GB | $0,41–0,64 | $0,23 | 24-GB-Klasse schnell |
| RTX 5090 | 32 GB | $0,69–0,89 | $0,69 | >24 GB ohne Kauf |
| RTX A6000 | 48 GB | $0,31–0,46 | $0,31 | große Modelle resident |
| RTX PRO 6000 (Blackwell) | 96 GB | ~$1,09–1,34 | — | 70B-Klasse resident |
| H100 | 80 GB | ~$2,00 | — | Trainieren, nicht unser Thema |

Preise schwanken mit der Nachfrage — **vor jeder Entscheidung live checken**
(vast.ai/pricing oder ein Preis-Tracker), diese Tabelle ist ein Snapshot.

## Break-even gegen lokale Hardware

Beispielrechnung auf Monatsbasis (730 h = 24/7; Arbeitsnutzung ≈ 176 h):

| Nutzungsprofil | Miete (4090, typ.) | Lokal (24-GB-Setup ~2.500 € Anschaffung) |
|---|---|---|
| 24/7-Dauerbetrieb | ~$470/Monat | nach ~5–6 Monaten günstiger, danach ~Strom |
| Werktage 8 h (~176 h) | ~$113/Monat ≈ €105 | nach ~2 Jahren günstiger |
| Gelegentliche Experimente (~20 h) | **~$13/Monat** | nie — Hardware liegt 99 % brach |

**Faustregel:** unter ~40 Stunden/Monat gewinnt Mieten klar; über ~100 Stunden/Monat
lohnt Kauf; dazwischen entscheidet Komfort (Cloud = null Wartung, jederzeit eine
größere Karte; lokal = Datenschutz, keine Daten-abfließ-Risiken, volle Flag-Kontrolle,
kein Vendor-Lock).

## Wann Mieten gewinnt

- **Ausprobieren vor dem Kauf:** erst einen Monat RTX-5090 mieten und das eigene
  Workload-Profil messen, dann die Kauf-Entscheidung
- **Spitzenbedarf:** das 70B-Modell für ein Wochenende statt 96-GB-Karte für 3.000 €
- **Kein Platz/Lärm/Netzwerkbau im eigenen Heim** (lokal = Tower, Storage, Parity-Jobs)
- **Cloud-Engine-Vergleich:** dieselbe GGUF auf CUDA vs. dem eigenen Backend messen

## Wann lokal gewinnt (und warum dieses Repo trotzdem lokal ist)

1. **Datenschutz ist das Hauptargument des lokalen Betriebs** (Kap. 1) — auf einer
   gemieteten Maschine verlassen Code, private Daten und API-Keys das Haus. Wer
   lokal betreibt *weil* Datenschutz zählt, kann mieten nur für datenfreie
   Experimente nutzen.
2. **Agent-Dauerbetrieb**: ein Coding-Agent, der stundenläuft, wird im Stundenmiete
   schnell teurer als abgeschriebene Hardware (s. Tabelle oben).
3. **Flag-Tiefe**: eigene Hardware = eigene Build-Flags, eigener Kernel-Stack, kein
   Host-Image, das dir die Stellschrauben wegkonfiguriert.
4. **Verfügbarkeit**: Community-Maschinen können weggebrochen werden (Host schaltet ab);
   die eigene Karte hängt nicht am Marktplatz.

## Praktische Hinweise für den Miet-Fall

- llama.cpp läuft dort mit denselben Flags wie lokal — die Tuning-Reihenfolge aus
  [ANLEITUNG.md](ANLEITUNG.md#phase-4--tuning-in-fester-reihenfolge-gate-je-schritt-warm-messen)
  gilt unverändert, nur `-t/-tb` (CPU-Kerne des Hosts) und der Backend-Typ (CUDA statt
  SYCL) ändern sich → KV-Quant-Ergebnisse **pro Backend neu messen**
- Institutionen/Communities mit kurzem Bedarf: Spot-Instanzen statt On-Demand
  (typisch 30–50 % günstiger, können aber vorzeitig enden — Checkpoints setzen)
- Storage und Traffic kosten extra: GGUFs müssen bei jedem neuen Server neu
  hochgeladen oder dort gezogen werden — bei 17-GB-Modellen und Drosselung gern
  eine Stunde „Reparatur" pro Instanzwechsel
