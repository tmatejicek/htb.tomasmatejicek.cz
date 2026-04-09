---
layout: post
author: Tomáš Matějíček
title: "Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji"
date: 2020-10-27
tags: passwords auth ssh winrm operations
---
## Úvod a kontext

Spousta útoků ve skutečnosti nestojí na nové zranitelnosti, ale na starém provozním zvyku: stejné heslo, passphrase nebo tajemství funguje na více místech zároveň. Jednou se objeví v `credentials.txt`, podruhé v databázové konfiguraci, potřetí v logu nebo ve starém image artefaktu. Samo o sobě to může vypadat jako lokální detail. Jakmile ale stejné tajemství otevře SSH, WinRM, Webmin nebo jiný administrativní kanál, hranice mezi aplikací a systémem se rozpadne.

To je přesně důvod, proč má password reuse smysl chápat jako samostatný bezpečnostní vzorec. Problém není jen ve slabém hesle. Problém je v tom, že jedno tajemství současně rozhoduje o více vrstvách prostředí, které by měly být oddělené.

## Proč je reuse tak nebezpečný

Bezpečnostní model mnoha systémů předpokládá, že jednotlivé vrstvy se brání nezávisle:

- webová aplikace má vlastní účty,
- databáze vlastní credentialy,
- SSH nebo WinRM samostatné přihlášení,
- admin nástroje typu Webmin, CMS nebo TeamViewer další oddělenou správu.

Jakmile se ale stejné heslo objeví ve dvou nebo více z nich, tento model přestává platit. Útočník nemusí exploitovat druhou vrstvu. Stačí mu jen pochopit, kde jinde stejné tajemství vyzkoušet.

Právě to dělá z password reuse jednu z nejpraktičtějších post-exploitation technik. Nízkoprivilegovaný nález v aplikaci se během chvíle může změnit v plnohodnotný shell nebo v administrativní přístup.

## Kde se reuse v praxi rodí

Stejné heslo se mezi vrstvami obvykle neobjeví náhodou. Důvody bývají velmi provozní:

- administrátor chce mít stejné heslo do více nástrojů,
- vývojář použije stejné tajemství pro aplikaci i lokální účet,
- bootstrap nebo instalační heslo zůstane v provozu příliš dlouho,
- passphrase ke klíči se znovu použije jako heslo uživatele,
- pomocný nástroj typu TeamViewer nebo Webmin sdílí stejné heslo jako systémový účet,
- totéž heslo se začne používat i pro RDP přístup přes [xfreerdp](/nastroje/xfreerdp),
- uniklá databázová nebo aplikační credential se z pohodlnosti znovu použije i jinde.

Všechny tyto situace vypadají na první pohled jako drobné provozní rozhodnutí. V útoku ale znamenají přesně to, že jedna nalezená hodnota může otevřít několik různých služeb.

## Jak reuse vypadá v konkrétních vzorech

### 1. Tajemství z webu otevře shell

Častý scénář je, že webová aplikace nebo její záloha vydá:

- `credentials.txt`,
- `.env`,
- `db.php`,
- nebo jiný konfigurační soubor.

V tu chvíli je potřeba položit si otázku, jestli nalezené heslo patří jen aplikaci, nebo i člověku či systémovému účtu. U více rozborů strojů právě tohle rozhodlo přechod z webového nálezu na SSH nebo `su`.

Typický příklad vypadá takto:

```php
$dbUser = 'theseus';
$dbPass = 'iamkingtheseus';
```

Samo o sobě to ještě není shell. Jakmile ale v systému existuje účet `theseus` a stejné nebo odvozené heslo funguje i lokálně, databázové tajemství přestává být jen databázové.

### 2. Tajemství z jedné služby otevře jiný admin kanál

Reuse nebývá jen mezi webem a SSH. Velmi častý je i mezi:

- uživatelským účtem a Webminem,
- lokálním heslem a WinRM,
- uloženou passphrase a administrativním rozhraním,
- klíčem z backupu a přihlášením do dalšího nástroje.

To je bezpečnostně nebezpečné právě proto, že admin nástroje mají obvykle širší dopad než první kompromitovaná vrstva. Útok pak nepokračuje exploitem, ale prostým přihlášením.

### 3. Provozní tajemství unikne z vedlejší vrstvy

Někdy ani nejde o heslo ve formuláři nebo configu, ale o artefakt, který k aplikaci přiléhá:

- log s nechtěným login pokusem,
- TeamViewer tajemství v registru,
- passphrase ke klíči nalezenému na disku,
- shell history z image vrstvy,
- starý backup s provozní poznámkou.

Taková hodnota může působit okrajově, ale v reálném řetězci často funguje jako klíč k úplně jiné službě.

## Jak tento vzorec vypadal v různých případech

Na [Admireru](/admirer) veřejně dostupné administrativní soubory vydaly seznam přístupových údajů a skutečná hodnota přišla až ve chvíli, kdy se jedno z hesel potvrdilo na SSH účtu `waldo`. Rozhodující nebyl samotný únik textového souboru, ale zjištění, že údaje nejsou omezené jen na FTP nebo WordPress.

Na [Magic](/magic) webový shell odhalil databázový účet a hodnoty z aplikační tabulky. Až jejich reuse na systémovém účtu `theseus` proměnil webový foothold v trvalý SSH přístup. Útočník v takové chvíli nepotřebuje další zranitelnost, jen správně přečíst vztah mezi aplikací a hostem.

Na [Remote](/remote) vedla záloha webu a provozní logy k heslu administrátora Umbraca. Po prvním footholdu se ale ukázalo, že lokálně uložené TeamViewer tajemství funguje i pro `Administrator` přes WinRM. Tady je krásně vidět, že reuse nemusí vypadat jako stejný login ve dvou formulářích. Může jít i o sdílené tajemství mezi desktopovým nástrojem a privilegovaným systémovým účtem.

Na [Postmanu](/postman) stál další případ na tom, že passphrase k nalezenému `id_rsa.bak` nebyla důležitá jen pro klíč samotný. Stejná hodnota otevřela i lokální účet a následně správu ve Webminu. Tím se z jednoho na první pohled vedlejšího artefaktu stal průchod hned přes několik vrstev.

Na [Cache](/cache) nešlo o reuse jedné databázové hodnoty mezi dvěma formuláři, ale o propojení několika vrstev. Heslo z `functionality.js` pomohlo přepnout se z OpenEMR webshellu na lokální účet `ash`, zatímco další tajemství z Memcached otevřelo `luffy` a nakonec i root přes `docker` skupinu.

Na [Delivery](/delivery) zase reuse nevypadal jako první zjevná slabina. Lokální konfigurace OsTicketu a Mattermostu daly dohromady seed `PleaseSubscribe!`, hint `Crack_The_MM_Admin_PW` a hash účtu `root`. Až jejich spojení ukázalo, že tematicky odvozené heslo funguje i na lokálním root účtu.

## Jak reuse disciplinovaně ověřovat

Password reuse se neověřuje chaotickým "zkusím to všude". Užitek přináší hlavně tehdy, když se testuje podle kontextu.

### Nejdřív určit, komu credential pravděpodobně patří

- je to lidský účet,
- služba,
- admin panel,
- deployment uživatel,
- lokální technický účet,
- nebo jen databázový login bez shellu.

### Pak hledat realistické cíle

- `/etc/passwd`, `net user`, home directory a lokální účty,
- SSH, WinRM, `su`, `runas`,
- CMS nebo administrační panel,
- Webmin, TeamViewer, interní nástroje,
- další služby zmíněné v configu nebo logu.

### A teprve potom zkoušet reuse

Rozumný postup tedy není "střílet heslo všude", ale spojit:

- nalezené jméno,
- konkrétní službu,
- a technický důvod, proč by reuse mohl dávat smysl.

Právě to odděluje užitečnou analýzu od hlučného credential sprayingu.

## Jaké artefakty reuse často prozradí

Při post-exploitation i při review se vyplatí cíleně hledat:

- `credentials.txt`, `.env`, `db.php`, `config.yml`,
- instalační a migrační skripty,
- logy neúspěšných i úspěšných loginů,
- záložní SSH klíče a jejich passphrase,
- desktopové a klientské nástroje ukládající tajemství,
- shell history a build artefakty,
- exporty admin nástrojů nebo support archivy.

Každý z těchto zdrojů může obsahovat tajemství, které se samo tváří lokálně, ale ve skutečnosti žije i jinde.

## Obrana a provozní hygiena

### 1. Každá vrstva má mít vlastní credential

Heslo do aplikace, heslo uživatele na SSH, heslo do Webminu a tajemství desktopového nástroje nesmějí být stejné. Jinak neexistuje skutečné oddělení vrstev.

### 2. Provozní tajemství považovat za privilegovaný materiál

TeamViewer heslo, passphrase k privátnímu klíči nebo tajemství z image vrstvy nejsou "vedlejší provozní údaje". Jsou to přihlašovací materiály se stejnou citlivostí jako běžné heslo.

### 3. Omezit, kde se vůbec dá stejná credential použít

Pokud účet nepotřebuje interaktivní login, neměl by ho mít. Pokud je heslo určeno pro aplikaci, nemá fungovat i na shellu nebo v admin nástroji.

### 4. Hlídání reuse zahrnout i do incident response

Při nálezu uniklého hesla nestačí změnit jednu službu. Je potřeba dohledat, kde všude stejná hodnota nebo její varianta funguje, a teprve potom mluvit o nápravě.

### 5. Logy a zálohy chránit stejně jako produkci

Mnoho reuse řetězců nezačíná ve veřejné aplikaci, ale v logu, backupu nebo support archivu. Pokud tyto artefakty obsahují hesla nebo provozní tajemství, jejich únik má stejnou hodnotu jako únik produkční databáze.

## Shrnutí klíčových poznatků

- Password reuse není jen slabé heslo, ale architektonický problém mezi více vrstvami prostředí.
- Jedna nalezená credential často nestačí sama o sobě. Rozhodující je pochopit, kde jinde může stejná hodnota dávat smysl.
- Reuse se často objevuje mezi webem, SSH, WinRM, Webminem, desktopovými nástroji a provozními artefakty.
- Obrana stojí na oddělení credentialů, omezení interaktivního loginu a na tom, že logy, zálohy a klientské artefakty nejsou brané jako druhotné.

## Co si odnést do praxe

- Když aplikace vydá heslo, neřešte jen to, co otevře uvnitř té aplikace. Ptejte se, jestli stejná hodnota nemíří i na shell, WinRM nebo jiný admin kanál.
- Password reuse je potřeba testovat cíleně podle kontextu účtu a služby, ne plošným sprayováním.
- Bezpečnostní hranice mezi aplikací, operačním systémem a správními nástroji existuje jen tehdy, když každá vrstva používá jiné credentialy a jiný model přístupu.
