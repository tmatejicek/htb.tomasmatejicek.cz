---
layout: post
author: Tomáš Matějíček
title: "Supply chain uvnitř firmy: interní package registry"
date: 2021-01-17
tags: supply-chain pypi packages registry development
---
## Úvod a kontext

Když se mluví o supply-chain útocích, pozornost často míří na veřejné ekosystémy jako PyPI, npm nebo RubyGems. Jenže podobně nebezpečný problém vzniká i uvnitř firmy. Interní package registry se často berou jako bezpečné právě proto, že jsou interní. Ve skutečnosti ale bývají ještě zrádnější:

- důvěřuje jim vývojový nebo provozní workflow,
- instalace z nich probíhá automaticky,
- a přístup do nich bývá chráněný slaběji než veřejná produkce.

SneakyMailer je přesně ten typ případu, kde registry sama není první foothold. Je až další vrstvou. Jakmile se útočník dostane k účtu, který smí do interního PyPI nahrávat, přestává jít o „repozitář balíčků“. Jde o execution kanál do důvěryhodného prostředí.

## O čem tenhle článek je a o čem není

Tenhle text není o dependency confusion vůči veřejnému internetu. Neřeší situaci, kdy build omylem stáhne balíček z veřejného PyPI místo interního.

Řeší jiný model:

- firma má vlastní registry,
- interní systémy jí důvěřují,
- a útočník získá možnost do ní nahrát nebo upravit balíček.

To je důležitý rozdíl. Tady problém nevzniká na hranici mezi interním a veřejným ekosystémem. Problém vzniká uvnitř organizace, protože interní důvěra se automaticky převádí na spuštění cizího kódu.

## Proč je interní registry tak silný pivot

Interní registry je nebezpečná tím, že propojuje dvě věci:

- write přístup na artefakt,
- a automatickou důvěru konzumenta.

Jakmile workflow dělá něco jako:

- `pip install`,
- `npm install`,
- `gem install`,
- nebo automatický fetch interního balíčku,

nepracuje už jen se statickými daty. Pracuje s objektem, který může během instalace:

- spustit hook,
- nainstalovat soubor do citlivé cesty,
- zapsat klíč nebo konfiguraci,
- nebo jinak změnit chování cílového systému.

To je přesně důvod, proč interní registry není „jen mirror“. Je to distribuční kanál pro kód.

## SneakyMailer: od `.pypirc` k cizímu SSH účtu

Na SneakyMailer se po prvním footholdu objevila interní služba:

```text
pypi.sneakycorp.htb
```

Současně se podařilo získat dva kritické podklady:

- `htpasswd` záznam pro účet `pypi`,
- a soubor `.pypirc`, který říkal, kam a jak se balíčky nahrávají.

```ini
[local]
repository: http://pypi.sneakycorp.htb:8080
username: pypi
password: soufianeelhaoui
```

To je zásadní moment. Tím se z obyčejného config souboru stává mapa supply-chain cesty:

1. existuje interní registry,
2. existuje účet s upload oprávněním,
3. existuje workflow, které z ní něco instaluje,
4. a někdo jiný této registry implicitně věří.

Právě poslední bod je klíčový. Balíček nemá hodnotu sám o sobě. Hodnotu má proto, že ho někdo jiný nainstaluje jako legitimní součást interního procesu.

## Proč nestačí říct „balíček obsahoval škodlivý setup.py“

To by bylo příliš úzké. Důležitý není konkrétní Python detail. Důležitý je obecný princip:

- systém přijímá balíček jako důvěryhodný artefakt,
- a při zpracování vykoná logiku, kterou útočník ovlivnil.

Ve SneakyMailer se to projevilo v `setup.py`, které při instalaci zapsalo útočníkův klíč do:

```text
/home/low/.ssh/authorized_keys
```

Tím vznikl přístup k účtu `low`. Ale stejný model může v jiných ekosystémech znamenat:

- spuštění postinstall hooku,
- vytvoření persistence,
- zápis do konfiguračního adresáře,
- nebo změnu produkčního chování aplikace.

Proto je lepší chápat interní registry jako execution trust boundary, ne jako prostý souborový sklad.

## Jak vypadá interní supply-chain důvěra v praxi

V běžném prostředí bývají přítomné tyto vazby:

- vývojář nebo servisní účet může nahrávat balíčky,
- jiný účet je automaticky instaluje,
- CI/CD nebo cron důvěřuje konkrétnímu internímu source,
- a nikdo aktivně neověřuje, zda je nový balíček legitimní změna, nebo cizí zásah.

Jakmile tahle důvěra existuje, útok se láme na jedné otázce:

- může útočník do této registry něco dostat.

Pokud ano, další krok už není exploit služby. Je to zneužití legitimního distribučního kanálu.

## Co při analýze hledat

Při pohledu na interní registry nebo lokální package workflow mají největší hodnotu tyto indicie:

### 1. Konfigurační soubory s přístupem do registry

Typicky:

- `.pypirc`,
- `.npmrc`,
- `.gemrc`,
- `pip.conf`,
- CI proměnné,
- nebo servisní configy v home adresáři.

Tyto soubory často neříkají jen „kam se připojit“. Říkají i:

- kdo tam smí nahrávat,
- jaký endpoint se používá,
- a jaké workflow na něj navazuje.

### 2. Účty, které mají publish oprávnění

Upload účet do interní registry je vysoce citlivý. Nemusí mít SSH ani sudo. Pokud ale jeho balíček někdo důvěryhodně instaluje, jde prakticky o nepřímou cestu ke kódu.

### 3. Automatická nebo pravidelná instalace

Největší riziko vzniká tam, kde:

- se balíčky instalují automaticky,
- aktualizace nejsou ručně schvalované,
- a instalující účet má hodnotnější oprávnění než účet, který publishuje.

### 4. Možnost write -> authenticate

SneakyMailer je hezký právě tím, že škodlivý balíček neřešil reverzní shell během instalace. Využil jednodušší a stabilnější variantu: zapsal klíč do `authorized_keys` a otevřel legitimní přihlášení.

To je důležitá obranná lekce. Supply-chain dopad není vždy vidět jako hlučný malware. Často je to tichá změna důvěryhodné konfigurace.

## Rozdíl proti článku o deployment trustu

Je užitečné tenhle článek oddělit od textu o repozitáři a deployment trustu.

Tam jde hlavně o to, že:

- working tree,
- Git historie,
- hooky,
- nebo deployment skripty

řídí produkční chování.

Tady je těžiště jinde:

- interní registry je distribuční kanál,
- balíček je důvěryhodný artefakt,
- a instalace je execution moment.

Jinými slovy: neřešíme „repo jako zdroj nasazení“, ale „registry jako zdroj důvěryhodného kódu“.

## Obrana a hardening

### 1. Považovat publish oprávnění za privilegovaný přístup

Účet, který smí do interní registry nahrávat balíčky, není obyčejný vývojářský účet. Je to účet s potenciálem ovlivnit cizí execution workflow.

### 2. Oddělit publish a consume identity

Když stejný nebo slabě chráněný účet:

- publikuje balíček,
- a jiný důvěryhodný proces ho bez dalšího ověření instaluje,

vzniká přímý supply-chain kanál.

### 3. Zavést schvalování, podpisy nebo jiné ověření artefaktů

Interní neznamená automaticky důvěryhodné. Registry by měla odlišovat:

- kdo smí publikovat,
- kdo schvaluje,
- jak se ověřuje původ,
- a jak se auditují změny.

### 4. Minimalizovat účinek instalačního procesu

Účet, který balíček instaluje, nemá mít zbytečně široká práva. Čím vyšší oprávnění při instalaci, tím vyšší dopad kompromitované registry.

### 5. Monitorovat nové nebo neobvyklé publikace

Zejména varovné jsou:

- nový balíček se jménem podobným internímu modulu,
- nečekaná nová verze,
- publish mimo běžný release cyklus,
- nebo publish z účtu, který se běžně používá jen ke čtení.

## Shrnutí klíčových poznatků

- Interní package registry není jen convenience vrstva pro vývoj. Je to důvěryhodný distribuční kanál pro kód.
- SneakyMailer ukazuje celý supply-chain řetězec: přístup do interního PyPI -> publish škodlivého balíčku -> automatické zpracování jiným účtem -> nový SSH přístup.
- Největší riziko neleží v konkrétním Python detailu typu `setup.py`, ale v tom, že publish účet může ovlivnit, co jiný systém důvěryhodně spustí nebo zapíše.
- Obrana musí interní registry chránit stejně vážně jako deployment nebo CI/CD.

## Co si odnést do praxe

- Kdo smí publikovat do interní registry, ten už v praxi často ovlivňuje cizí kódové nebo autentizační workflow.
- Interní balíček není bezpečný jen proto, že nepřišel z veřejného internetu.
- Pokud instalace balíčku probíhá automaticky a s vyššími právy, publish účet do registry je třeba považovat za citlivý privilegovaný přístup.
