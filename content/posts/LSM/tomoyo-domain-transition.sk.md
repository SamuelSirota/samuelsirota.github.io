---
title: "Tomoyo LSM prechody domén"
date: "2025-03-03"
ShowToc: true
---

Ako TOMOYO rozhodne, či proces môže spustiť program (prostredníctvom `execve()`) a prepínať domény

**Príklad:** Proces v `<kernel> /sbin/init` chce spustiť `/usr/bin/calc`

## Krok 1: Získanie názvu cesty programu

- Získajte úplnú cestu programu `[candidate]`
- môže byť symlink
- **Príklad:**

  ```plaintext
  /usr/bin/calc → [candidate] = „/usr/bin/calc“
  ```

## Krok 2: Kontrola smernice agregátora

- Skontrolujte politiku výnimiek pre `agregator [candidate] [pathname]`
- Ak sa nájde, nahraďte `[candidate]` za `[pathname]`
- **Príklad:**

  ```plaintext
  aggregator /usr/bin/calc /bin/calculator → [candidate] = „/bin/calculator“
  ```

- Agregátor zoskupuje podobné programy; ak žiadny nie je, ponechá pôvodný `[candidate]`
  
## Krok 3: Kontrola smernice `file execute`

- Vyhľadajte v politike aktuálnej domény (napr. `<kernel> /sbin/init`) príkaz `file execute [candidate] ...`
- **Možnosti (nastaví `[destination]`, prejde na krok 5):**
- `file execute [candidate] keep` → zostať v `<kernel> /sbin/init`
- `file execute [candidate] child` → `<kernel> /sbin/init /usr/bin/calc`
- `file execute [candidate] reset` → `<kernel> /usr/bin/calc`
- `file execute [candidate] initialize` → `<kernel> /usr/bin/calc` (menný priestor root)
- `file execute [candidate] parent` → `<kernel>` (nadradená doména)
- `file execute [candidate] [názov domény]` → vlastná doména (napr. `<kernel> /usr/bin/game`)
- `file execute [candidate] [pathname]` → `<kernel> /sbin/init /some/path`
- `file execute [candidate]` → bez modifikátora, prejdite na krok 4
- **Fail:** Žiadne pravidlo + vynucovací mód → Deny
- **Príklad:**  

  ```plaintext
  file execute /usr/bin/calc reset → [destination] = <kernel> /usr/bin/calc
  ```

## Krok 4: Určenie predvoleného prechodu domény

- Ak sa v kroku 3 nenastavil `[destination]`, skontrolujte politiku výnimiek v poradí:

1. `reset_domain [candidate] z [source] (alebo „any“) + no_reset_domain` → `<kernel> /usr/bin/calc`, skok na Krok 5
2. `initialize_domain [candidate] z [source] (alebo „any“) + no no_initialize_domain` → `<kernel> /usr/bin/calc`, skok na krok 5
3. `keep_domain [candidate] z [source] (alebo „any“) + no no_keep_domain` → `<kernel> /sbin/init`, skok na krok 5
4. **Default** → `<kernel> /sbin/init /usr/bin/calc`

- **Príklad:**  

  ```plaintext
  initialize_domain /usr/bin/calc → [destination] = <kernel> /usr/bin/calc
  ```

- Náhradný plán, ak bol krok 3 nejasný

## Krok 5: Vytvorenie cieľovej domény

- Nastavte `[destination]` z kroku 3 alebo 4
- **Prípady zlyhania:**
  - Pravidlo kroku 3, nemožno vytvoriť → Odmietnuť
  - `reset_domain` → Deny
  - Krok 4 predvolený + vynucovací režim → Deny
- Krok 4 predvolený + iný režim → vykonávanie pokračuje, ale zostáva v aktuálnej doméne, označte smernicu `transition_failed`

## Krok 6: Kontrola premenných prostredia

- Overte premenné prostredia (napr. `PATH`) v `[destination]`
- ak je >1 env var zamietnutá + vynucovací režim → Deny
- **Príklad:**

  ```plaintext
  LD_PRELOAD“ zamietnutý v ‚<kernel> /usr/bin/calc‘ → Deny ak vynucovanie
  ```

## Krok 7: Kontrola interpretov

- Ak program potrebuje interpreter (napr. `/bin/bash` pre skript), skontrolujte oprávnenie na čítanie v `[destination]`
- ak je čítanie pre interpret zamietnuté + vynucovanie režimu → Deny
- **Príklad:**

  ```plaintext
  /usr/bin/calc potrebuje /bin/bash, čítanie v /bin/bash zamietnuté → zlyhanie, ak vynucovanie
  ```

## Krok 8: Spustenie programu

- Spustite program, ak je úspešný, prepnite na `[destination]`
- **Príklad:** `/usr/bin/calc` spustí sa → proces sa presunie do `<kernel> /usr/bin/calc`
- Úspech = prechod dokončený

## Diagram prechodov domény

```plaintext
Začiatok: <kernel> /sbin/init
1. Získajte /usr/bin/calc
2. Agregátor? → Nie
3. Spustenie súboru?
    - Áno → Nastaviť [destination] → Prejsť na 5
    - Nie → 4. Predvolené nastavenia?
        - reset_domain → <kernel> /usr/bin/calc
        - initialize_domain → <kernel> /usr/bin/calc
        - keep_domain → <kernel> /sbin/init
        - Predvolené nastavenie → <kernel> /sbin/init /usr/bin/calc
5. Vytvoriť [destination]
    - Áno → 6. Env vars v poriadku?
        - Áno → 7. Interpreter v poriadku?
            - Áno → 8. Spustenie a prechod ÚSPECH
            - Nie → DENY (enforcing mode)
        - Nie → ODMIETNÚŤ (enforcing mode)
    - Nie → ODMIETNÚŤ alebo zostať v aktuálnej doméne
```

## Príklad scenára

- `<kernel> /sbin/init`, chce prejsť na `/usr/bin/calc`
- Politika: `file execute /usr/bin/calc inicializovať`, enforcing mode
- **Kroky:**
  1. `[candidate] = „/usr/bin/calc“`
  2. Žiadny agregátor
  3. `initialize` → `[destination] = <kernel> /usr/bin/calc`, skok na 5
  4. Vytvoriť `<kernel> /usr/bin/calc`
  5. Env vars v poriadku → Áno
  6. Žiadne problémy s interpreterom → Áno
  7. Spustenie → Prechod na `<kernel> /usr/bin/calc`

## Pripojenie poznámok

- **Kľúčové pojmy:**
  - `[candidate]`: Cesta k programu
  - `[destination]`: Nová doména
  - `[current_domain]`: doména (napr. `<kernel> /sbin/init`)
  - `[current_namespace]`: Koreň ako `<kernel>`
- **Priorita:** Krok 3 > Krok 4 (špecifické pravidlá prekonávajú predvolené)
- **Režim vynucovania:** Striktné zakázanie pri zlyhaní
