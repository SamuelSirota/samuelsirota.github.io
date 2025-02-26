---
title: "Tomoyo LSM inštalácia"
date: "2025-02-25"
ShowToc: true
---

## Inštalácia TOMOYO na Ubuntu 24.04.01

Najprv skontrolujeme, či má jadro povolené TOMOYO:

```bash
grep tomoyo_write_inet_network /proc/kallsyms
```

Výpis by mal byť takýto:

```bash
0000000000000000 T tomoyo_write_inet_network
```

Výpis znamená, že tomoyo LSM je zakompilovaný do jadra, nemusíme ho aktivovať v jadre, čo nám uľahčuje prácu.

## Inštalácia závislostí

Pred stiahnutím a kompiláciou nástrojov používateľského priestoru TOMOYO musíme nainštalovať niektoré knižnice.

```bash
sudo apt-get -y install wget patch gcc make libncurses-dev
```

## Konfigurácia jadra

Túto časť môžeme preskočiť, pretože naše jadro už má TOMOYO aktivované. Ak by tomu tak nebolo, museli by sme stiahnuť zdrojové kódy jadra a nakonfigurovať ho. Táto časť je obsiahnutá v [dokumentácii TOMOYO](https://tomoyo.sourceforge.net/2.6/chapter-3.html.en#3.1.3).

## Inštalácia nástrojov používateľského priestoru

Najprv si stiahneme nástroje zo sourceforge a overíme podpis pomocou kumaneko-key.
Potom môžeme rozbaliť archív a skompilovať nástroje.

```bash
wget https://sourceforge.net/projects/tomoyo/files/tomoyo-tools/2.6/tomoyo-tools-2.6.1-20241111.tar.gz
wget https://sourceforge.net/projects/tomoyo/files/tomoyo-tools/2.6/tomoyo-tools-2.6.1-20241111.tar.gz.asc
wget https://tomoyo.sourceforge.net/kumaneko-key
gpg --import kumaneko-key
gpg --verify tomoyo-tools-2.6.1-20241111.tar.gz.asc
tar -zxf tomoyo-tools-2.6.1-20241111.tar.gz
cd tomoyo-tools
make -s USRLIBDIR=/usr/lib
sudo make -s USRLIBDIR=/usr/lib install
```

## Inicializácia konfigurácie

Tu pridáme nástroje tomoyo userspace tools do našej cesty PATH:

```bash
export PATH=$PATH:/usr/sbin
```

Nie je to isté, ale pre istotu som to pridal do svojho `.bashrc`.

Ďalej sme inicializovali politiku:

```bash
sudo /usr/lib/tomoyo/init_policy
```

Mali by sme dostať viacero OK výstupov.

## Konfigurujeme zavádzač

```bash
sudo nano /etc/default/grub
```

Tu upravíme riadok `GRUB_CMDLINE_LINUX` tak, aby obsahoval `security=tomoyo`.

```bash
GRUB_CMDLINE_LINUX="quiet splash security=tomoyo“
```

Potom aktualizujeme grub.

```bash
sudo update-grub
```

## Reštartujeme

Teraz reštartujeme systém:

```bash
sudo reboot
```

Po reštarte môžeme skontrolovať, či je TOMOYO povolené. Pomocou tohto príkazu:

```bash
cat /sys/kernel/security/lsm
```

Výstup by mal vyzerať asi takto:

```bash
lsm=capability,yama,loadpin,safesetid,selinux,tomoyo
```

Máme povolený a spustený systém TOMOYO. Teraz ho môžeme začať konfigurovať.
