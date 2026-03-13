---
layout: post
author: Tomáš Matějíček
title: "Traverxec"
date: 2021-01-27
tags: linux ssh web exploit privesc
---
## Úvod a kontext

Traverxec je postavený na méně obvyklém webserveru Nostromo. Právě to je na něm didakticky zajímavé: první část útoku nestojí na známém Apache nebo nginx workflow, ale na správném rozpoznání konkrétní verze `nostromo 1.9.6` a její RCE chyby.

Po webovém footholdu následuje pěkný lokální pivot. Konfigurace Nostroma prozradí existenci chráněné domácí zóny uživatele `david`, odkud se dá stáhnout záloha SSH identity. Root část je pak klasická GTFOBins situace kolem `journalctl`, ale důležité je nejdřív pochopit, odkud se vůbec bere možnost spouštět jej přes `sudo`.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejdřív ověřuji veřejné služby.

```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1
80/tcp open  http    nostromo 1.9.6
```

Rozhodující je právě identifikace Nostroma. U méně rozšířených serverů má smysl téměř okamžitě hledat známé CVE, protože patch management u nich bývá slabší.

## Analýza zjištění

### RCE v Nostromu

Verze `1.9.6` byla zranitelná vůči path traversal/RCE chybě (`CVE-2019-16278`). Ta umožní získat shell jako `www-data`.

Jakmile shell běží, má smysl podívat se přímo do konfigurace webserveru:

```bash
cat /var/nostromo/conf/nhttpd.conf
cat /var/nostromo/conf/.htpasswd
```

Konfigurace ukazovala dvě podstatné věci:

- server používá HTTP basic auth z `.htpasswd`,
- má zapnuté `homedirs` s veřejnou cestou `public_www`.

`htpasswd` obsahoval hash pro uživatele `david`:

```text
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

Ten šel cracknout:

```bash
/usr/sbin/john Traverxec-htpasswd.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```text
Nowonly4me
```

### Soukromá zóna uživatele `david`

Díky `homedirs_public public_www` bylo možné přistupovat do části webového prostoru patřící uživateli `david`. Tam ležela záloha SSH identity:

```text
http://10.10.10.165/~david/protected-file-area/backup-ssh-identity-files.tgz
```

Archiv obsahoval `id_rsa`, ale klíč byl chráněný passphrase. Tu šlo zpracovat přes `ssh2john.py` a následně cracknout:

```bash
/usr/share/john/ssh2john.py Traverxec-ssh/id_rsa > Traverxec-ssh/id_rsa.john
/usr/sbin/john Traverxec-ssh/id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```

```text
hunter
```

Tím vznikla plnohodnotná SSH identita uživatele `david`.

## Získání přístupu

### SSH jako `david`

Po odemknutí klíče už následovalo stabilní přihlášení:

```bash
ssh -i Traverxec-ssh/id_rsa david@10.10.10.165
```

Pak bylo možné potvrdit user flag:

```bash
cat user.txt
```

```text
__CENSORED__
```

## Eskalace oprávnění

### Proč je důležitý helper skript

Lokální enumerace ukázala pomocný skript, který vypisoval statistiky Nostroma. Klíčová byla jeho poslední řádka:

```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

To je přesně ten typ detailu, který snadno zapadne. `david` neměl obecné `sudo`, ale měl možnost spustit `journalctl` nad jednotkou `nostromo.service`. Jakmile je `journalctl` spuštěný přes `sudo`, je potřeba myslet i na jeho pager.

### Únik do shellu z `less`

`journalctl` vypíše výstup přímo do terminálu jen tehdy, pokud se vejde na obrazovku. Pokud ne, použije `less`. A `less` umí shell escape pomocí `!`.

Proto stačilo:

1. zmenšit terminál tak, aby se i pět řádků nevešlo bez stránkování,
2. spustit příkaz s `journalctl`,
3. v `less` zadat `!/bin/sh`.

Výsledkem je shell stále běžící v kontextu roota, protože `journalctl` byl spuštěn přes `sudo`.

Pak už bylo možné přečíst `root.txt`.

## Shrnutí klíčových poznatků

- První foothold zde stál na správném rozpoznání Nostromo 1.9.6 a jeho známé RCE.
- Konfigurace webserveru měla po footholdu vyšší hodnotu než běžná systémová enumerace, protože odkryla `htpasswd` i veřejné homediry uživatele `david`.
- Root část nevyžadovala nový exploit, ale pochopení toho, že `journalctl` pod `sudo` může přepnout do pageru a odtud spustit shell.

## Co si odnést do praxe

- Méně obvyklé servery a aplikace je potřeba aktivně inventarizovat. Pokud organizace provozuje software mimo hlavní proud, často zaostává i jeho patchování.
- Konfigurace webserveru může po footholdu prozradit další cestu útoku: auth soubory, exportované homediry nebo neveřejné aliasy mají často větší hodnotu než samotný webový obsah.
- `sudo` výjimky pro zdánlivě neškodné nástroje jako `journalctl`, `less` nebo `man` je potřeba posuzovat i podle jejich interních funkcí. Pager s `!` je v praxi shell escape.
