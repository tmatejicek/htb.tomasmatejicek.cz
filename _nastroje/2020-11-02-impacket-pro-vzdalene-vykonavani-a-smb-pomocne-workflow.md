---
layout: post
author: Tomáš Matějíček
title: "Impacket pro vzdálené vykonávání a SMB pomocné workflow"
date: 2020-11-02
tags: nastroje windows smb winrm lateral-movement execution
---
## Úvod a kontext

Jiná část Impacketu dává smysl až ve chvíli, kdy už máte použitelný účet nebo hash a potřebujete z něj udělat vzdálené vykonávání nebo technický SMB mezikrok. Sem patří hlavně `psexec.py`, `wmiexec.py`, `smbexec.py` a `impacket-smbserver`. Nejde o jednu kategorii exploitu. Jde o čtyři různé způsoby, jak převést existující identitu do praktické práce s cílovým Windows hostem.

Na [Bankrobberu](/bankrobber) se tyto utility objevují jako přirozené pokračování po získání účtu. Na [Driveru](/driver) a [Resolute](/resolute) je zase dobře vidět role `impacket-smbserver` jako jednoduchého doručovacího kanálu. Tato rodina je praktická hlavně proto, že řeší úplně obyčejnou, ale kritickou otázku: jak z existujících creds udělat použitelnou interakci s hostem.

## Co tyhle utility v praxi řeší

Prakticky pokrývají dvě sousední potřeby:

- vzdálené spuštění příkazu nebo shellu na Windows hostu,
- a pomocné SMB workflow, například doručení DLL, skriptu nebo binárky.

To je důležité oddělit od jiných AD nástrojů. Tady už nejde o enumeraci nebo dump tajemství. Jde o to, jak s už získanou identitou nebo hashem skutečně pracovat.

## Nejčastější scénáře využití v tomto projektu

### `psexec.py`, `wmiexec.py`, `smbexec.py`: převod účtu na vzdálené vykonávání

Na [Bankrobberu](/bankrobber) se právě tato trojice přirozeně objevuje jako sada kandidátů, jak z nového přístupu udělat shell nebo příkazový kanál.

Každý z nástrojů řeší podobný cíl trochu jinak:

- `psexec.py` je silný tehdy, když chcete robustnější vzdálené spuštění přes službu,
- `wmiexec.py` je často lehčí a méně invazivní pro jednorázové příkazy,
- `smbexec.py` stojí víc na SMB a dočasném servisním workflow.

Praktický rozdíl nebývá "který je nejlepší obecně", ale "který se hodí na konkrétním hostu, s konkrétním účtem a konkrétním typem oprávnění".

Na hostech, kde je otevřený WinRM, může být pohodlnější [Evil-WinRM](/nastroje/evil-winrm). Když ale WinRM není k dispozici nebo potřebujete jiný typ přístupu, sahá se právě po této Impacket trojici.

### `impacket-smbserver`: rychlé doručení souboru cílovému hostu

Na [Driveru](/driver) je role `impacket-smbserver` velmi čistá. Pro PrintNightmare bylo potřeba vystavit škodlivou DLL přes SMB share:

```bash
impacket-smbserver data .
```

Na [Resolute](/resolute) se stejný pattern objevuje u DLL pro DNS plugin. V obou případech je hlavní hodnota nástroje v jednoduchosti: během několika vteřin vytvoříte SMB share, ze kterého si cílový host může načíst soubor.

To je přesně důvod, proč má `impacket-smbserver` vysokou praktickou hodnotu i mimo "velké" AD řetězce. Je to malý technický pomocník, který výrazně zjednoduší doručení dalšího kroku.

### Volba mezi WinRM, SMB exec a pomocným share

V místních rozborech se pěkně opakuje jeden pattern:

- když existuje WinRM, často vítězí [Evil-WinRM](/nastroje/evil-winrm),
- když potřebujete jednorázové spuštění bez WinRM, nastupují `psexec.py`, `wmiexec.py` nebo `smbexec.py`,
- když jen doručujete DLL nebo binárku, stačí `impacket-smbserver`.

Právě to je užitečný praktický rámec. Tyto utility neřeší stejný problém, i když všechny spadají do jednoho toolkitu.

## Kdy jsou tyto utility nejlepší volba

Tahle rodina dává největší smysl tehdy, když:

- už máte použitelný účet nebo hash,
- potřebujete vzdáleně spustit příkaz,
- WinRM není ideální nebo dostupný,
- nebo je potřeba cíli rychle vystavit soubor přes SMB.

Naopak nedávají smysl jako náhrada za první enumeraci nebo za privilege analysis. Když ještě nevíte, zda účet vůbec k něčemu vede, je brzy je vytahovat.

## Nejčastější chyby v praxi

### Zaměnění všech tří exec utilit za totéž

`psexec.py`, `wmiexec.py` a `smbexec.py` mají podobný výsledek, ale jinou provozní stopu a jinou vhodnost podle cíle. Vyplatí se vybírat podle kontextu, ne podle zvyku.

### Ignorování jednoduššího kanálu

Když už běží WinRM, bývá pohodlnější [Evil-WinRM](/nastroje/evil-winrm). Když jde jen o doručení souboru, často stačí `impacket-smbserver` bez složitějšího exec workflow.

### Přeskakování technického mezikroku

Na [Driveru](/driver) a [Resolute](/resolute) je hezky vidět, že SMB helper může být stejně důležitý jako samotný exploit. Bez něj by cílový host neměl odkud načíst DLL nebo jiný payload.

## Související články v projektu

- Rozbory: [Bankrobber](/bankrobber), [Driver](/driver), [Resolute](/resolute)
- Techniky: [Windows lokální privesc přes služby, drivery a spooler](/techniky/windows-lokalni-privesc-pres-sluzby-drivery-a-spooler)
- Nástroje: [Impacket](/nastroje/impacket), [Evil-WinRM](/nastroje/evil-winrm), [smbclient](/nastroje/smbclient)

## Co si odnést do praxe

- Tato rodina Impacket utilit řeší okamžik po získání identity, kdy potřebujete vzdálené vykonávání nebo SMB pomocný kanál.
- `psexec.py`, `wmiexec.py` a `smbexec.py` nejsou zaměnitelné jen podle jména; liší se stylem práce i vhodností pro konkrétní host.
- `impacket-smbserver` je malý, ale velmi praktický helper pro doručení DLL, binárky nebo jiného souboru tam, kde cílový host umí číst SMB share.
