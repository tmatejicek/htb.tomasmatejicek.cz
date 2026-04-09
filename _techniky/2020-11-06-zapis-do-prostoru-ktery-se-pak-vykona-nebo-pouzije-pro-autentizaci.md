---
layout: post
author: Tomáš Matějíček
title: "Zápis do prostoru, který se pak vykoná nebo použije pro autentizaci"
date: 2020-11-06
tags: file-write execution authentication webroot ssh deployment
---
## Úvod a kontext

Jedna z nejčastějších analytických chyb po footholdu vypadá nevinně:

- `můžu tam zapisovat, ale ještě to není shell`.

V mnoha prostředích je to naopak téměř hotový kompromis. Jakmile útočník dokáže zapsat do prostoru, který se později:

- vykoná,
- načte jako deploy artefakt,
- použije jako autentizační materiál,
- nebo zpracuje privilegovaná automatika,

nejde už o „jen write“. Jde o přímou nebo téměř přímou cestu k dalšímu přístupu.

Právě tenhle pattern spojuje na první pohled různé případy:

- upload ASPX do webrootu,
- deploy WAR do Tomcatu,
- zápis do `authorized_keys`,
- změnu souboru, který si stáhne produkční aplikace,
- nebo writable skript spouštěný přes `sudo`.

## O čem tenhle článek je a o čem není

Tenhle text se schválně překrývá s několika jinými tématy jen částečně.

Není to článek o nebezpečných upload funkcích. Tam je hlavní otázka, jak upload projde validační logikou a kdo ho interpretuje.

Není to ani článek o repozitáři a deployment trustu. Tam jde o to, že Git nebo historie konfigurace řídí důvěryhodný provozní proces.

Tady je hlavní abstrakce jiná:

- pokud mám write do místa, které jiná komponenta považuje za důvěryhodný vstup, write samotný už skoro znamená execute nebo authenticate.

## Proč je „jen write“ tak podceňované

Důvod je jednoduchý. Lidé intuitivně hodnotí zápis jako slabší oprávnění než spuštění. Jenže to platí jen u inertních dat. Jakmile zapisovaný soubor později čte jiný proces s jinou logikou nebo jinými právy, write se mění na:

- odložené spuštění kódu,
- odložený login,
- nebo odložený privesc.

Prakticky je užitečné ptát se ne:

- `můžu tam psát`,

ale:

- `co se s tím místem stane potom`.

## Varianta 1: write -> execute přes webroot

[Devel](/devel) je nejčistší ukázka. Anonymous FTP samo o sobě není shell. Rozhodující je až to, že FTP míří do prostoru, který servíruje IIS:

```text
ftp-anon: Anonymous FTP login allowed
iisstart.htm
```
Praktickou roli samotného protokolu rozebírám i v článku [FTP](/nastroje/ftp).

Jakmile lze do stejného prostoru zapsat ASPX soubor, další krok už není kreativní. Webserver soubor vykoná. Tady se write oprávnění okamžitě mění v execute.

To je důležitá lekce: write do webrootu není „souborový přístup“. Je to nepřímé RCE.

## Varianta 2: write -> execute přes oficiální deploy kanál

[Jerry](/jerry) stojí na trochu jiné verzi téhož principu. Tomcat Manager sice funguje jako legitimní administrativní rozhraní, ale z bezpečnostního pohledu je to pořád write do prostoru, který se pak vykoná.

Nahraný `WAR` není statický soubor. Je to artefakt, který aplikační server:

- rozbalí,
- nasadí,
- a spustí.

To je přesně ten důvod, proč veřejně přístupný Manager se slabým heslem znamená prakticky okamžité převzetí hostu. Write do deployment prostoru je už execute.

## Varianta 3: write -> authenticate přes `authorized_keys`

[Zetta](/zetta) ukazuje jinou, velmi praktickou variantu. Rsync export domácího adresáře `roy` dovolil nejen čtení, ale i zápis. Tím bylo možné podstrčit:

```text
/home/roy/.ssh/authorized_keys
```

To je přesný příklad patternu `write -> authenticate`. Útočník nepotřebuje shell v době zápisu. Potřebuje jen zapsat do místa, které SSH později přijme jako důvěryhodný autentizační materiál.

Stejný princip se opakuje i jinde:

- `.k5login`,
- `.ssh/config`,
- token cache,
- service config,
- nebo jakýkoli soubor, který určuje, kdo a jak se smí přihlásit.

## Varianta 4: write -> execute přes aplikaci nebo repo

Na [Bitlabu](/bitlab) je dobře vidět, že write nemusí směřovat jen do webrootu nebo deployment rozhraní. Jakmile kompromitovaný uživatel může změnit soubor:

```php
profile/index.php
```

a běžící aplikace tuto změnu převezme, write se znovu mění v execute.

To už je blízké deployment trustu, ale důležitý je stejný sjednocující bod: útok nevyžaduje zranitelnost interpretru. Stačí změnit to, co interpret důvěryhodně načte a spustí.

## Varianta 5: write -> privileged execute přes pomocný skript

[Nibbles](/nibbles) přidává ještě jednu vrstvu. Uživatel `nibbler` mohl přes `sudo` spustit:

```text
/home/nibbler/personal/stuff/monitor.sh
```

Jestliže je tento soubor současně zapisovatelný, nejde už o „omezené sudo na konkrétní skript“. Jde o write do prostoru, který se později vykoná s vyššími právy.

To je velmi důležitý pattern při lokální analýze:

- writable skript,
- writable service config,
- writable plugin,
- nebo writable hook

často znamená totéž jako přímé spuštění kódu pod cizí identitou, jen s malým zpožděním.

## Jak tenhle pattern poznat systematicky

Když někde najdeš write přístup, má smysl projít tyto otázky:

### 1. Kdo ten soubor nebo adresář později čte

- webserver,
- aplikační server,
- SSH,
- cron,
- `sudo`,
- CI/CD,
- nebo jiná privilegovaná automatika.

### 2. Jde o data, nebo o řídicí vstup

Bezpečnostně je nejrizikovější vše, co určuje:

- co se má spustit,
- kdo se smí přihlásit,
- jaký plugin se načte,
- odkud se vezme deployment,
- nebo jaký skript poběží s vyššími právy.

### 3. Jak krátká je cesta od write k dopadu

Někdy je dopad okamžitý:

- webroot,
- Tomcat deploy,
- plugin load.

Jindy je odložený:

- příští cron běh,
- další SSH login,
- příští deploy,
- další `sudo` spuštění.

### 4. Je write lokální, síťový nebo přes aplikaci

Tohle je důležité pro obranu. Pattern je stejný, ale vstup se liší:

- FTP do webrootu,
- rsync do home,
- upload do Manageru,
- Git commit do běžící aplikace,
- nebo lokální přepis skriptu.

## Co podobné případy spojuje

Na první pohled jde o různé technologie, ale všechny stojí na stejné důvěře:

- někdo předpokládá, že zapisované místo je „správné“,
- a pozdější komponenta už z něj bez další pochybnosti čte nebo spouští obsah.

Právě proto write často přeskakuje mezikrok, který by jinak byl potřeba:

- není nutný samostatný exploit,
- není nutná krádež hesla,
- není nutné obcházet paměťové ochrany.

Stačí dodat obsah na správné místo.

## Obrana a hardening

### 1. Oddělit write prostor od execute a authenticate prostoru

Nejzákladnější pravidlo zní: uživatel nesmí zapisovat tam, odkud se později spouští kód nebo odkud se bere autentizační materiál.

### 2. Nepoužívat writable uživatelské cesty pro privilegované workflow

`sudo` skript, cron job nebo deployment krok nesmí číst z cesty, kterou může měnit neprivilegovaný účet.

### 3. Chránit deploy a auth soubory jako řídicí rovinu

`authorized_keys`, plugin adresáře, webrooty, hooky, playbooky a deployment artefakty nejsou obyčejná data. Jsou to řídicí vstupy bezpečnostního modelu.

### 4. Auditovat write oprávnění stejně pečlivě jako execute oprávnění

V řadě případů má write do citlivého místa téměř stejný dopad jako přímé execute. Kontrola oprávnění proto nesmí končit u toho, kdo smí spustit binárku.

### 5. Monitorovat změny v citlivých cestách

Zvlášť důležité je hlídat:

- nové soubory ve webrootu,
- změny `authorized_keys`,
- nové nebo změněné hooky,
- změny skriptů spouštěných přes `sudo` nebo cron,
- a neočekávané deploy artefakty.

## Shrnutí klíčových poznatků

- Write oprávnění nejsou samy o sobě neškodné. Pokud míří do prostoru, který se později vykoná nebo použije pro autentizaci, jsou často už téměř ekvivalentem shellu nebo loginu.
- Devel ukazuje `write -> execute` přes webroot, Jerry přes deploy rozhraní a Zetta `write -> authenticate` přes `authorized_keys`.
- Bitlab a Nibbles doplňují, že stejný pattern funguje i u aplikací, repozitářů a pomocných skriptů spouštěných s vyššími právy.
- Obrana stojí hlavně na oddělení write prostoru od execute/auth prostoru a na pečlivém auditu toho, čemu prostředí bez dalšího věří.

## Co si odnést do praxe

- Když někde najdeš write přístup, neptej se jen `co tam mohu nahrát`. Ptej se hlavně `kdo to potom spustí nebo použije k přihlášení`.
- `authorized_keys`, webroot, plugin adresář, deploy artefakt nebo sudo skript jsou z bezpečnostního pohledu stejná rodina problémů: write do důvěryhodného vstupu.
- V mnoha reálných prostředích není nejkratší cesta k shellu memory corruption ani exploit. Je to prostě správně zvolený soubor na správném místě.
