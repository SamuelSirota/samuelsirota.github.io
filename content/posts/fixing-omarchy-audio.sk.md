---
date: '2025-12-06'
title: 'Oprava Omarchy Audia'
tags: ["omarchy","arch","audio","linux"]
showToc: true
---

Nedávno som mal problémy na mojom Arch Linux omarchy setupe. Systém nevedel zaznamenať audio jack a preto som nevedel používať moje káblové slúchadlá.
Pokúšal som sa to opraviť už nejaký čas a konečne sa mi to podarilo opraviť.

## Nájdenie audio kodeku

Tento príkaz nám vypíše zariadenie ktoré počítač používa na kódovanie audia.

```bash
cat /proc/asound/card0/codec* | grep -i Codec
```

Značku a model tohto zariadenia môžeme použiť na nájdenie známych problémov a riešení ku konkrétnym zariadeniam.
Môj počítač používa `Conexant CX118800`, zistil som že tento kodek má množstvo známych problémov obzvlášť pri používaní Linuxu.

## Inštalácia nástrojov

Nástroje ktoré budeme potrebovať sú `hdajackretask` na premapovanie pinov, `alsamixer` a `alsactl` na monitorovanie a `wpctl` na manažment výstupov.

```bash
sudo pacman -S alsa-tools alsa-utils
```

## Použitie nástrojov

```bash
alsactl monitor
```
Tento príkaz monitoruje všetky zmeny uskutočňujúce sa na hardware, ukazuje zmeny hlasitosti, stlmenie, pripojenie/odpojenie audio zariadení a podobne. Pomocou neho môžeme zistiť či firmware zaznamená pripojenie slúchadiel.

```bash
wpctl status
```
Tento príkaz zobrazuje aktuálny stav objektov v našom PipeWire multimedia frameworku.

## Mapovanie pinov pre audio zariadenia

Na opravenie nerozoznania audio jacku našim firmware použijeme `hdajackretask` na prepísanie pinov v kodeku.

```bash
sudo hdajackretask
```

Po otvorení nástroja `hdajackretask` vyberieme v menu kodek v našom prípade `Conexant CX118800`. Ďaľej môžeme vidieť viacero pinov ktoré nie sú rozpoznané našim systémom, nastavíme ich ako potrebujeme. Zaškretneme možnosť Advanced override pre viacero možností.

1. Pin ID 0x16, popis tothto pinu píše: Black Headphone, Left side. Toto nám indikuje čo to je a tiež kde sa nachádza.
   - Zaškrtneme Override checkbox
   - `Connectivity`, vyberieme možnosť Jack
   - `Device`, vyberieme možnosť Headphone
   - `Jack`, vyberieme možnosť 3.5 mm
   - `Jack Selection`, vyberieme Present pre aktiváciu

2. Pin ID 0x17, popis tothto pinu píše: Internal Speaker, Rear side. 
   - Zaškrtneme Override checkbox
   - `Connectivity`, vyberieme možnosť Internal
   - `Device`, vyberieme možnosť Speaker
   - `Jack Selection`, vyberieme Not present

3. Pin ID 0x19, popis tothto pinu píše: Black Mic, Left side. Ak máme iba jeden audio jack to znamená že nastavíme jack pre detekciu slúchadiel aj mikrofónu. 
   - Zaškrtneme Override checkbox
   - `Connectivity`, vyberieme možnosť Jack
   - `Device`, vyberieme možnosť Microphone
   - `Jack`, vyberieme možnosť 3.5 mm
   - `Jack Selection`, vyberieme Present pre aktiváciu

4. Pin ID 0x1a, popis tothto pinu píše: Internal Mic, Top.
   - Zaškrtneme Override checkbox
   - `Connectivity`, vyberieme možnosť Intenal
   - `Device`, vyberieme možnosť Microphone
   - `Jack Selection`, vyberieme Not present

Po nastavení všetkých pinov stlačíme Apply now a otestujeme či všetko správne funguje, skontrolujeme aj výstupy z monitorovacích nástrojov.

Ak všetko správne funguje môžeme nainštalovať boot override a spraviť tieto zmeny trvalé. Potom môžeme reštartovať a všetko by malo fungovať správne.