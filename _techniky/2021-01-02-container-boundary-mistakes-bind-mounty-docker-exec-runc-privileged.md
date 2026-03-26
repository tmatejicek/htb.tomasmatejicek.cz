---
layout: post
author: Tomáš Matějíček
title: "Container boundary mistakes: bind mounty, `docker exec`, `runc`, `privileged`"
date: 2021-01-02
tags: containers docker runc bind-mount privileged privesc
---
## Úvod a kontext

Kontejner se často popisuje jako bezpečnostní hranice. V praxi je to ale jen část pravdy. Kontejner umí dobře oddělit procesy a filesystem, pokud je kolem něj rozumně navržené celé prostředí. Jakmile se do hry přidají bind mounty z hosta, přehnaná práva runtime nebo delegovaná správa přes `docker exec`, začne se hranice rozpadat.

Ready, TheNotebook a Registry ukazují tři různé způsoby, jak k tomu dochází. Jednou je problém přímo v `privileged: true`, podruhé v hostem mountnuté cestě a právu spouštět `docker exec`, potřetí v tom, že image registry a build artefakty vydají tajemství, která přesahují samotný kontejner. Nejde tedy jen o klasické "escape z Dockeru". Jde o širší chybu návrhu: systém se tváří, že kontejner odděluje role, ale okolní provozní vrstva to sama popře.

## Co vlastně kontejner odděluje a co už ne

Samotný kontejner typicky odděluje:

- procesy,
- namespacy,
- část filesystemu,
- síťový prostor,
- a v lepším případě i capabilities.

To ale ještě neznamená, že odděluje:

- tajemství uložená v image,
- hostem mountnuté adresáře,
- runtime správu přes Docker CLI,
- nebo privilegia, která orchestrace kontejneru přidělí navíc.

Když se řekne "kontejnerová hranice", vyplatí se proto ptát ve čtyřech vrstvách:

1. co je uvnitř image,
2. co je mountnuté z hosta,
3. co umí dělat runtime a kdo ho smí ovládat,
4. s jakými právy kontejner skutečně běží.

## Nejčastější chyby, které hranici bortí

### 1. Bind mount mění zápis v kontejneru na efekt na hostu

To, že aplikace běží v kontejneru, ještě neznamená, že zapisuje jen do kontejnerového filesystemu. Pokud je upload adresář nebo jiná cesta bind mountnutá z hosta, může útočníkův zápis uvnitř kontejneru skončit jako host-level artefakt.

### 2. `privileged` z kontejneru dělá téměř hosta

`privileged: true` není běžná optimalizace. Je to zásadní oslabení hranice. Root uvnitř takového kontejneru má často velmi krátkou cestu k host filesystemu, zařízením nebo dalším nízkoúrovňovým primitivům.

### 3. `docker exec` pod `sudo` je správa hosta, ne "jen kontejner"

Kdo smí přes `sudo` spouštět `docker exec`, nedostává jen omezené právo "podívat se do kontejneru". Dostává přístup k vysoce privilegované runtime vrstvě hosta. Na starších verzích může jít o přímý exploit chain, ale i bez něj je to stále velmi silné oprávnění.

### 4. Image a registry patří do stejného trust modelu

Hranice se může rozpadnout ještě před spuštěním kontejneru. Jakmile registry nebo image vrstvy vydají shell historii, privátní klíče nebo provozní skripty, izolace runtime už situaci nezachrání. Útočník dostane tajemství, která platí mimo samotný kontejner.

## Jak tenhle vzorec vypadal v praxi

### TheNotebook: bind mount a host-level vykonání PHP

TheNotebook je velmi dobrý příklad toho, že mountovaná cesta je součást bezpečnostní hranice. Aplikace běžela v Dockeru, ale upload adresář byl bind mountnutý z hosta a nginx na hostu přímo zpracovával `.php` soubory z této cesty. To změnilo význam uploadu: nahraný `rev.php` se nespouštěl uvnitř kontejneru, ale na hostitelském systému.

To je důležitý detail. Nešlo o "escape z kontejneru" v tradičním smyslu. Šlo o to, že hranice byla sama navržená tak, že zápis v kontejneru okamžitě získal efekt na hostu.

### TheNotebook podruhé: `docker exec` a starý `runc`

Stejný stroj přidal i druhou variantu. Uživatel `noah` mohl přes `sudo` spouštět:

```text
(ALL) NOPASSWD: /usr/bin/docker exec -it webapp-dev01*
```

Na hostu běžela stará verze Dockeru a `runc`, takže šlo zneužít `CVE-2019-5736`. Podstatná lekce ale není samotné číslo CVE. Podstatné je, že `docker exec` je správní operace nad runtime hosta. Jakmile k ní má běžný uživatel přístup, je hranice mezi hostem a kontejnerem už velmi tenká.

### Ready: root v kontejneru + `privileged: true`

Ready ukazuje nejpřímočařejší rozpad hranice. Foothold vznikl v GitLab kontejneru, poté se z `gitlab.rb` získalo heslo a útočník se stal rootem uvnitř kontejneru. Samotný root v kontejneru by ještě automaticky neznamenal host kompromis. Rozhodující byl až obsah `docker-compose.yml`:

```yaml
privileged: true
```

To otevřelo přímou cestu k host filesystemu, například přes mount disku:

```bash
mount /dev/sda2 /mnt
```

Na tomhle příkladu je krásně vidět, že problém neležel jen v aplikaci. Skutečný dopad určila až provozní konfigurace kontejneru.

### Registry: image vrstvy a tajemství přesahující kontejner

Registry není klasický container escape, ale je důležité ho do této rodiny zahrnout. Veřejně dostupná Docker registry vydala image vrstvu, v níž byly:

- shellové historické artefakty,
- soukromý klíč,
- a jeho passphrase.

Tím se ukázalo, že hranice kontejneru nezačíná až při běhu procesu. Začíná už při build pipeline, image layer a tom, co se do ní dostane. Pokud image obsahuje host-relevantní tajemství nebo provozní klíče, izolace runtime je už jen částečná obrana.

Právě proto dává smysl číst Registry jako třetí typ container boundary chyby: nikoli únik z runtime, ale únik z container lifecycle.

## Jak podobné chyby hledat po footholdu

Po shellu v aplikaci nebo v kontejneru se vyplatí systematicky projít čtyři vrstvy.

### 1. Ověřit, zda opravdu běžíte v kontejneru

Pomohou obvyklé indicie:

- `.dockerenv`,
- minimalistický userspace,
- nečekaná síťová konfigurace,
- nebo cesty a proměnné typické pro Docker.

### 2. Zjistit, co je mountnuté z hosta

Právě bind mount často rozhoduje, zda zápis v kontejneru zůstane lokální, nebo zasáhne hosta. Hodnotné jsou hlavně:

- upload cesty,
- konfigurační adresáře,
- logy,
- sockety,
- a zálohy.

### 3. Projít runtime oprávnění

Kritické jsou hlavně:

- `privileged: true`,
- přístup k `docker.sock`,
- `sudo docker exec`,
- a staré nebo zranitelné runtime komponenty.

### 4. Podívat se na image a build artefakty

Když je dostupná registry, image nebo lokální cache vrstev, má smysl číst i je. Často vydají:

- shell historii,
- klíče,
- interní konfigurační soubory,
- nebo detaily, které přímo propojí kontejner s hostovou identitou.

## Jak si to nesplést s příbuznými tématy

Tento okruh se přirozeně překrývá s jinými články, ale je dobré držet rozdíly.

- Oproti článku o localhost službách jde tady primárně o hranici mezi hostem a container runtime, ne jen o interní porty.
- Oproti článku o `sudo` nad container nástroji se tu řeší širší model: mounty, orchestraci, image a privilegia kontejneru jako takového.
- Oproti storage a registry článku je důraz na to, jak container ecosystem ruší oddělení rolí, ne jen na to, že někde unikla data.

## Obrana a bezpečnější návrh

### 1. Bind mounty používat minimálně a cíleně

Každý hostem mountnutý adresář je součást bezpečnostní hranice. Pokud aplikace zapisuje do mountu a host stejnou cestu interpretuje jiným runtime, je to vysoce rizikový návrh.

### 2. `privileged: true` brát jako výjimku nejvyšší závažnosti

Privilegovaný kontejner nemá být pohodlný default. Má být výjimečný, zdokumentovaný a pravidelně auditovaný stav.

### 3. Nedávat běžným účtům správu Docker runtime

`docker exec`, `docker run`, přístup k `docker.sock` a podobné operace mají bezpečnostní význam blízký rootovi. Obranně je potřeba auditovat je stejně přísně.

### 4. Image vrstvy chránit jako produkční tajemství

Registry, build cache a image historie nesmějí obsahovat klíče, hesla, shell historii ani jiné provozní artefakty, které překračují roli samotné aplikace.

### 5. Threat model stavět přes celý container lifecycle

Bezpečnost kontejneru nezačíná při `docker run`. Začíná už v image, pokračuje přes mounty a orchestrace a končí až u runtime práv na hostu.

## Shrnutí klíčových poznatků

- Kontejner není automatická bezpečnostní hranice. Reálný dopad určuje až kombinace mountů, runtime práv a práce s image.
- TheNotebook ukazuje host-level efekt bind mountu i nebezpečí `docker exec`, Ready přímý dopad `privileged: true` a Registry to, že image vrstvy samy mohou narušit oddělení mezi aplikací a hostem.
- Při auditu kontejnerů je potřeba dívat se nejen na proces uvnitř, ale i na celý ekosystém kolem něj.

## Co si odnést do praxe

- Když vidíte kompromitovaný kontejner, první otázka nemá být "jak z něj utéct?", ale "co je sem mountnuté, s jakými právy běží a kdo smí ovládat runtime?".
- Bind mount, `privileged` a `docker exec` jsou tři velmi odlišné mechanismy, ale všechny dokážou zrušit bezpečnostní přínos kontejneru.
- Image a registry patří do stejného trust modelu jako runtime. Pokud build artefakty uniknou nebo nesou cizí tajemství, hranice je narušená ještě dřív, než kontejner vůbec startuje.
