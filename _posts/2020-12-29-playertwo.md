---
layout: post
author: Tomáš Matějíček
title: "PlayerTwo"
date: 2020-12-29
tags: linux rce ssh php exploit enumeration
---
## Úvod a kontext

PlayerTwo spojuje dvě odlišné disciplíny. User část je hlavně o aplikačním výzkumu: Twirp RPC, `.proto` definice, zneužití TOTP workflow a chybně ověřovaný firmware. Root část je naopak čistě lokální binární exploit proti SUID utilitě `Protobs`.

Didakticky je na něm cenné právě tohle oddělení. První polovina není o náhodném brute force, ale o tom, jak z veřejně vystavené API dokumentace odvodit celý autentizační řetězec. Druhá polovina už vyžaduje obrácené inženýrství a pochopení heap chyb v programu běžícím s root právy.

## Počáteční průzkum

### Otevřené služby

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```

```text
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open  http    Apache httpd 2.4.29
8545/tcp open  http    twirp
```

Vedle portů bylo hned důležité i rozdělení aplikace na dva hosty:

- `player2.htb`
- `product.player2.htb`

Na prvním z nich se podařilo najít veřejně dostupný protobuf soubor:

```text
http://player2.htb/proto/generated.proto
```

## Analýza zjištění

### Twirp `GenCreds` a přístup do aplikace

Soubor `generated.proto` popisoval službu `Auth` a metodu `GenCreds`. Už nebylo potřeba hádat názvy endpointů ani strukturu dat. Stačilo sestavit správný Twirp POST request:

```bash
curl --request POST --location "http://player2.htb:8545/twirp/twirp.player2.auth.Auth/GenCreds" \
  --header "Content-Type: application/json" \
  --data '{}'
```

Volání vracelo validní kombinace uživatelských jmen a hesel, například:

```text
snowscan / ze+EKe-SGF^5uZQX
```

Tato pověření fungovala na přihlašovací stránce `product.player2.htb`, ale přihlášení se zastavilo na druhém faktoru.

### Zneužití TOTP backup kódu

V aplikaci bylo patrné, že vedle jednorázového kódu podporuje i backup codes. Současně byl veřejně dostupný endpoint `/api/totp.php`. S platnou session cookie tak dávalo smysl zkoumat, jakou `action` očekává.

Rozhodující požadavek vypadal takto:

```bash
curl --location "http://product.player2.htb/api/totp.php" \
  --header "Content-Type: application/json" \
  --cookie "PHPSESSID=..." \
  --data '{"action":"backup_codes"}'
```

Odpovědí byl záložní kód pro právě přihlášeného uživatele. To dokončilo autentizační tok a zpřístupnilo produktovou část webu.

Metodicky je to důležitý moment: 2FA zde nepadlo prolomením TOTP algoritmu, ale tím, že aplikace sama zpřístupnila nouzový kanál bez správné autorizace.

### Firmware a rozbitá kontrola podpisu

Po přihlášení byla dostupná dokumentace Protobs, firmware balíček a testovací stránka `/protobs/`, která měla ověřovat podepsaný firmware v sandboxu.

Analýza `Protobs.bin` ukázala, že binárka obsahuje několik volání `system()` a že v ní jde změnit jeden krátký příkaz bez zásadního rozbití struktury souboru. Praktický exploit stál na tom, že:

- firmware šel binárně upravit,
- původní podpis zůstal přiložený,
- a server přesto balíček přijal jako validní.

To znamená, že kontrola podpisu neověřovala celý obsah firmwaru tak, jak měla. Upravený firmware pak při testu spustil útočníkův příkaz a poskytl shell jako `www-data`.

### MQTT jako zdroj SSH klíče

Po footholdu už nebyl další důležitý směr web, ale lokální MQTT broker Mosquitto běžící jen na localhostu:

```text
/usr/sbin/mosquitto -c /etc/mosquitto/mosquitto.conf
```

Po přihlášení jako `www-data` šlo sledovat interní zprávy pomocí [mosquitto_sub](/nastroje/mosquitto-sub):

```bash
mosquitto_sub -h localhost -t '$SYS/#'
```

Na tématu `$SYS/internal/firmware/signing` se periodicky objevoval privátní klíč používaný k podepisování firmwaru. Ten byl zároveň SSH klíčem uživatele `observer`.

## Získání přístupu

### SSH jako `observer`

Jakmile byl klíč z MQTT k dispozici, dalo se přejít na stabilní systémový přístup:

```bash
ssh -i id_rsa observer@player2.htb
```

Tím se otevřela i user část:

```bash
cat user.txt
```

```text
__CENSORED__
```

## Eskalace oprávnění

### SUID `Protobs` a heap exploit

V `/opt/Configuration_Utility/` ležela SUID binárka `Protobs` spolu s vlastní `libc.so.6` a `ld-2.29.so`. To je silná indicie, že privesc nepovede přes `sudo`, ale přes binární chybu přímo v utilitě.

Reverzní analýza ukázala dvě důležité vlastnosti:

- při mazání a znovuvytváření konfigurací šlo vyvolat double free nad popisem konfigurace,
- práce s délkou popisu dovolovala NULL-byte overflow do velikosti sousedního heap chunku.

Samostatně by to na moderní glibc nestačilo, ale v kombinaci šlo:

1. získat leak adres z libc přes unsorted bin,
2. obejít tcache ochrany změnou velikosti chunku,
3. provést tcache poisoning a přepsat `__free_hook` adresou `system`.

Praktické spuštění bylo nejsnazší přes lokální wrapper, který interaktivní binárku vystavil na TCP port:

```bash
mkfifo /tmp/p; nc -lp 4444 < /tmp/p | /opt/Configuration_Utility/Protobs > /tmp/p &
```

Po odpálení exploitu nad touto instancí už stačilo nechat binárku zavolat `free()` nad řetězcem `/bin/sh` a vznikl root shell. Tím se dokončil i přístup k `root.txt`.

## Shrnutí klíčových poznatků

- PlayerTwo začíná únikem aplikační dokumentace: veřejný `.proto` soubor prakticky popsal, jak získat přihlašovací údaje z Twirp API.
- Druhý faktor padl na špatně chráněných backup codes, ne na slabině TOTP samotného algoritmu.
- Firmware testovací workflow dovolilo spustit upravený kód navzdory deklarovanému podpisu.
- User část vznikla až přes MQTT únik privátního klíče a root část pak přes heap exploit v SUID utilitě `Protobs`.

## Co si odnést do praxe

- RPC a protobuf definice patří mezi citlivé artefakty. Jakmile jsou veřejné, výrazně zkracují čas od reconu k funkčním požadavkům.
- Nouzové cesty v autentizaci, jako jsou backup codes nebo recovery API, musejí být chráněné stejně přísně jako hlavní login flow. Jinak znehodnotí celý 2FA model.
- Kontrola digitálního podpisu musí ověřovat přesně ten obsah, který se následně spouští. „Podpis existuje“ není totéž jako „obsah je integritně chráněný“.
- Interní message brokery a technická témata typu `$SYS/#` často nesou tajemství, která aplikace nikdy neměla zveřejnit. Monitoring a automatizace nesmějí distribuovat privátní klíče ani jiné citlivé artefakty.
- SUID binárky nad vlastní business logikou jsou vysoce rizikové. Jakmile kombinují správu heapu, vlastní datové struktury a root práva, je jejich audit mnohem důležitější než u běžných utilit.
