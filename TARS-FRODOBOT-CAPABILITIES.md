# TARS FRODOBOT — Kompletny prehlad schopnosti
> Verzia: 2.0 | Posledna aktualizacia: 2026-03-07
> GitHub: https://github.com/Patkopatko/tars-frodobot

---

## Robot

**Model:** FrodoBot EarthRover Mini+
**Identifikator:** `frodobot_pkjmes`
**Ovladanie:** Agora RTM (peer message `{linear, angular, lamp}`)
**Video:** Agora RTC — UID 1000 (predna kamera), UID 1001 (zadna kamera)
**Dashboard:** `http://localhost:8001/dashboard`

---

## SCHOPNOSTI — prehlad

| # | Nazov | Klaves/API | Stav |
|---|-------|-----------|------|
| S-01 | Manualny pohyb | WASD / sipky | OK |
| S-02 | Regulacia rychlosti | + / - | OK |
| S-03 | Lampa | L | OK |
| S-04 | Nudzove zastavenie | SPACE | OK |
| S-05 | Live telemetria | automaticky | OK |
| S-06 | TARS VISION — detekcia objektov | V | OK |
| S-07 | Meranie vzdialenosti od objektov | automaticky (Vision) | OK |
| S-08 | Sledovanie pohybu objektov (trend) | automaticky (Vision) | OK |
| S-09 | Popis obsahu zaberov (LLaVA AI) | automaticky (Vision) | OK |
| S-10 | TARS EXPLORER — autonomna exploracia | E | OK |
| S-11 | Vyhybanie sa prekazkam | automaticky (Explorer) | OK |
| S-12 | Robot State API (vzdialene ovladanie) | REST endpoints | OK |

---

## S-01 — Manualny pohyb

Ovladanie pohybu v realnom case cez klavesnicu.

| Klaves | Akcia |
|--------|-------|
| W / sipka hore | Pohyb dopredu |
| S / sipka dole | Pohyb dozadu |
| A / sipka vlavo | Otocenie vlavo |
| D / sipka vpravo | Otocenie vpravo |
| Kombinacia | Diagonalny pohyb (napr. W+D = dopredu-vpravo) |

Prikaz `{linear, angular, lamp}` posielany cez Agora RTM kazdu 100ms (10 Hz).

---

## S-02 — Regulacia rychlosti

Nastavenie maximalnej rychlosti pohybu pocas jazdy.

| Klaves | Akcia |
|--------|-------|
| + | Zvysenie o 0.05 m/s |
| - | Znizenie o 0.05 m/s |

**Rozsah:** 0.05 m/s (plazenie) az 0.50 m/s (maximum)
**Default:** 0.20 m/s
**Vizualizacia:** Farebny slider v pravom paneli (modra = pomaly, oranzova, cervena = rychly)

---

## S-03 — Lampa

Zapnutie/vypnutie predneho svetla robota.

| Klaves | Akcia |
|--------|-------|
| L | Toggle lampa ON/OFF |

Indikator: `LAMPA: ON/OFF` v bottom bare (zlta = zapnuta).

---

## S-04 — Nudzove zastavenie

Okamzite zastavenie vsetkej aktivity robota.

| Klaves | Akcia |
|--------|-------|
| SPACE | STOP — zastavi pohyb + vypne Explorer |

Vymaze aktivne klavesy, vypne Explorer ak bezi, posle `{linear:0, angular:0}`.

---

## S-05 — Live telemetria

Robot posiela stav kazdu ~500ms cez Agora RTM. Dashboard zobrazuje v realnom case:

| Velicina | Popis |
|----------|-------|
| Bateria | Stav nabitia v % (zelena >30%, oranzova >15%, cervena <15%) |
| Napatie | Napatie baterie vo Voltoch |
| Prud | Odber pradu v mA |
| Rychlost | Aktualna rychlost v m/s |
| Orientacia | Smer natocenia robota v stupnoch |
| GPS signal | Sila GPS signalu |
| Vibracia | Intenzita vibracii (detekcia nerovneho terenu) |
| WiFi signal | 5-pruhovy indikator sily signalu |

**Session timestamp:** `SES HH:MM:SS` v topbare — potvrdzuje ze token je cerstvy.
Agora tokeny expiruju ~24h. Priznak expiracie: cierne kamery + RTC ERR.
Riesenie: restart SDK + novy tab (nie refresh).

---

## S-06 — TARS VISION — detekcia objektov

Real-time detekcia objektov cez AI na NVIDIA DGX Spark.

| Klaves | Akcia |
|--------|-------|
| V | Zapni/vypni TARS VISION overlay |

**Pipeline:**
```
Browser  JPEG frame (kazdu 120ms)
      -> Spark 10.168.127.220:8010/detect
      -> YOLO11x (detekcia) + ByteTrack (tracking)
      -> Canvas overlay s ramcekami + labelmi
```

**AI modely na Sparku (NVIDIA GB10 Blackwell):**
- YOLO11x — detekcia 80+ tried objektov, ~14 FPS
- ByteTrack — persistentne ID pre kazdy objekt napriec framemi
- Depth Anything V2 Metric Indoor — skutocna vzdialenost v metroch
- LLaVA:7b — popis obsahu kazdeho detekovanaho objektu

**Vystup na canvas:**
- Farebne ramceky s rohovou grafikou (L-tvar v kazdom rohu, tech styl)
- Label: `trieda konfidencia% | X.Xm`
- Sipka trendu: hore = priblizuje sa, dole = vzdialuje sa
- Pulzujuci cerveny glow pri priblizujucom objekte

---

## S-07 — Meranie vzdialenosti od objektov

Kazdy detekhovany objekt ma priradenu realnu vzdialenost v metroch.

**Metoda:** Depth Anything V2 Metric Indoor — monokularna hlbkova mapa z jedneho JPEG snimku.
**Sampling:** Medianov patch 5x5 pixelov v strede bounding boxu (robustne voci sumu).
**Presnost:** Kalibracia pre interiery, rozsah ~0.3m az 20m.

**Farebne kodovanie ramcekov podla vzdialenosti:**

| Farba | Vzdialenost | Interpretacia |
|-------|-------------|---------------|
| Cervena | menej ako 1.0 m | Bezprostredna blizkost — nebezpecna zona |
| Oranzova | 1.0 az 2.5 m | Blizkost — opatrnost |
| Zelena | 2.5 az 5.0 m | Bezpecna vzdialenost |
| Modra | viac ako 5.0 m | Vzdialeny objekt |

---

## S-08 — Sledovanie pohybu objektov (trend)

ByteTrack sleduje kazdy objekt napriec framemi a vypocitava trend pohybu.

| Trend | Sipka | Vizual | Popis |
|-------|-------|--------|-------|
| approaching | hore | Cerveny pulzujuci glow, blikajuci ramcek | Objekt sa priblizuje k robotovi |
| receding | dole | Modry, poloprisvitny ramcek | Objekt sa vzdialuje |
| stable | — | Standardny plny ramcek | Objekt stoji na mieste |

**Algoritmus:** Delta plochy bounding boxu medzi framemi — vacsi box = blizsi objekt.

---

## S-09 — Popis obsahu zaberov (LLaVA AI)

Pre kazdy detekhovany objekt generuje LLaVA:7b textovy popis co vidno v ramceku.

**Priklady:**
- `potted plant` — "Green leaves on tree."
- `vase` — "Blurry gray curtains, obscured view of palm tree behind them"

**Zobrazenie:** Maly sivy text pod hlavnym labelom v ramceku.
**Vykon:** LLaVA bezi paralelne s YOLO na GB10 Blackwell.

---

## S-10 — TARS EXPLORER — autonomna exploracia

Robot autonomne preskumava priestor bez zasahu cloveka.

| Klaves | Akcia |
|--------|-------|
| E | Toggle Explorer ON/OFF |
| SPACE | Zastavi Explorer (nudzovo) |

**Podmienka:** Explorer funguje LEN ked je VISION zapnuta (potrebuje depth_m pre kazdy objekt).

**Stavovy automat:**
```
MOVING  ---- prekaska najdena ---> AVOIDING
  ^                                    |
  +-------- volna cesta najdena -------+
```

| Stav | Indikator | Spravanie |
|------|-----------|-----------|
| PRESKUMAVAM | zelena blikajuca | Ide dopredu rychlostou 0.10 m/s |
| VYHYBAM SA | oranzova rychlo blikajuca | Otaca sa na mieste, hlada volny smer |
| CAKAM NA VISION | oranzova | Stoji, caka kym Vision zacne dodavat detekcie |
| OFF | seda | Deaktivovany |

---

## S-11 — Vyhybanie sa prekazkam

Integrovane do Exploreru — robot automaticky reaguje na detekhovane prekazky.

**Parametre:**

| Parameter | Hodnota | Popis |
|-----------|---------|-------|
| OBSTACLE_DEPTH | 2.5 m | Hranica — blizsi objekt = prekazka |
| OBSTACLE_ZONE | 50% sirky framu | Sledovana zona (cely stred) |
| EXPLORER_SPEED | 0.10 m/s | Rychlost pri volnej ceste |
| AVOID_ANGULAR | 0.35 rad/s | Rychlost otacania pri vyhybani |
| MAX_AVOID_MS | 10 000 ms | Po 10s otacania v jednom smere -> flip smeru |

**Logika vyhybania:**
1. Detekovany objekt blizsi ako 2.5m v centralnej zone -> AVOIDING
2. Robot sa otaca (vlavo alebo vpravo)
3. Ak po 10s nenasiel volny smer -> otoci smer rotacie (umoznuje ~360 stupnov)
4. Ked centralna zona volna -> spat do MOVING

**Znamy limit:** Prazde steny bez objektov YOLO nedetekcuje -> depth_m pre ne nie je k dispozicii.
Buduci fix: center-forward depth sample bez bbox zavislosti (roadmapa F-05).

---

## S-12 — Robot State API (vzdialene ovladanie)

REST endpointy na SDK serveri (`localhost:8001`) pre integraciu s TARS OpenClaw.
Umoznuje ovladanie robota cez Telegram / WhatsApp.

| # | Endpoint | Metoda | Popis |
|---|----------|--------|-------|
| E-01 | /robot/state | GET | Celkovy stav: explorer on/off, posledna scena |
| E-02 | /robot/explore | POST | `{"enabled": true/false}` — zapni/vypni Explorer |
| E-03 | /robot/stop | POST | Nudzove zastavenie + RTM stop prikaz |
| E-04 | /robot/scene | GET | Posledne YOLO detekcie (co robot vidi) |
| E-05 | /robot/scene | POST | Dashboard sem posiela detekcie po kazdom frame |

**Tok prikazu (planhovany):**
```
Ty: "TARS, co vidis?"
  -> OpenClaw skill: GET /robot/scene
  -> {"boxes": [{"label":"person","depth_m":1.8}, {"label":"chair","depth_m":3.2}]}
  -> TARS: "Vidim osobu vo vzdialenosti 1.8m a stolicku vo vzdialenosti 3.2m."

Ty: "TARS, spust exploraciu"
  -> OpenClaw skill: POST /robot/explore {"enabled": true}
  -> Robot zacne autonomne preskumavat miestnost
```

**Sync:** Dashboard polling `/robot/state` kazde 2s — ak TARS zmeni stav, dashboard reaguje automaticky.

---

## Architektura systemu

```
+---------------------------------------------------------------+
|  TARS (OpenClaw / Telegram / WhatsApp)                        |
|  "co vidis?" / "spust exploraciu" / "zastav"                  |
+----------------------------+----------------------------------+
                             | REST API
                             v
+---------------------------------------------------------------+
|  SDK Server  localhost:8001  (Python FastAPI + uvicorn)       |
|  /robot/state  /robot/explore  /robot/stop  /robot/scene      |
+--------+-------------------------------------+----------------+
         | HTML (dashboard)                    | polling 2s
         v                                     v
+-------------------+              +---------------------------+
|  Browser          |              |  Dashboard JS             |
|  dashboard.html   |<-------------|  explorerToggle           |
+--------+----------+              |  visionLoop               |
         |                         |  pollRobotState           |
         | Agora RTC (video)        +-------------+------------+
         | Agora RTM (control)                    |
         v                          JPEG frame    v
+-------------------+              +---------------------------+
|  FrodoBot         |              |  NVIDIA DGX Spark         |
|  EarthRover Mini+ |              |  10.168.127.220:8010      |
|  frodobot_pkjmes  |              |  YOLO11x + ByteTrack      |
+-------------------+              |  Depth Anything V2        |
                                   |  LLaVA:7b                 |
                                   +---------------------------+
```

---

## Spustenie systemu

```bash
# 1. SDK (Mac Studio)
source ~/frodoenv/bin/activate
cd ~/earth-rovers-sdk
python -m uvicorn main:app --port 8001

# 2. Spark (skontroluj ci bezi)
curl http://10.168.127.220:8010/health

# 3. Dashboard — vzdy NOVY TAB (nie refresh!)
http://localhost:8001/dashboard
```

**Checklist po otvoreni:**
1. `SES HH:MM:SS` v topbare = aktualni cas (token cerstvy)
2. Kamery maju video (nie NO SIGNAL)
3. `Agora: CONNECTED + RTM OK` v bottom bare
4. Stlac `V` — pockat 3-4s (Spark CUDA warm-up), zezlenanie = OK
5. Stlac `E` — explorer ON

---

## Roadmapa

| # | Faza | Nazov | Popis |
|---|------|-------|-------|
| F-03 | Faza 3 | LLaVA Scene Feed | Celkovy popis sceny kazde 2-3s ako text overlay v dashboarde |
| F-04 | Faza 4 | TensorRT optimalizacia | 300+ FPS inferencia na GB10 Blackwell |
| F-05 | Faza 5 | Depth pre prazdne steny | Center-forward depth bez bbox zavislosti — detekovanie sten |
| F-06 | Faza 6 | Visual Servoing | `goto?target=chair` — najdi objekt a pribliz sa k nemu |
| F-07 | Faza 7 | OpenClaw FrodoBot skill | Plna integracia: Telegram/WhatsApp -> robot |

---

Dokument: 2026-03-07 | TARS FRODOBOT v2.0
