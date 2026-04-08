---
layout: post
author: Tomáš Matějíček
title: "Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects"
date: 2020-10-29
tags: active-directory windows dcsync gmsa relay acl
---
## Úvod a kontext

V Active Directory se často hledá jednoduchá hranice: buď je někdo `Domain Admin`, nebo není. V praxi je to výrazně složitější. Doménová kompromitace často nevzniká z přímého členství v privilegované skupině, ale z méně nápadných práv, která se v prostředí nahromadila kvůli provozu: delegované ACL, možnost relaynout autentizaci do LDAP, právo číst heslo gMSA, přístup k deleted objects nebo synchronizační účet, který drží dešifrovatelné tajemství.

Forest, Intelligence, Cascade, APT a Monteverde ukazují různé varianty stejného problému. Útočník nemusí dostat "admin účet" klasickou cestou. Stačí mu identita nebo servisní kontext, který umí:

- změnit oprávnění v adresáři,
- číst tajemství jiného účtu,
- vyžádat tiket jménem privilegované identity,
- nebo sáhnout na historický nebo synchronizační artefakt, který by v doméně vůbec neměl být dosažitelný.

## Co znamená delegované právo v AD

Delegované právo je prakticky každá schopnost, která není vidět jako přímé členství v silné skupině, ale přesto mění bezpečnostní stav domény. Může jít například o:

- ACL dovolující změnit jiný objekt,
- rozšířené právo k replikaci adresáře,
- právo číst heslo gMSA,
- přístup k citlivému historickému objektu,
- nebo administrativní roli v pomocné synchronizační komponentě.

To je důležité i metodicky. Když při review sleduješ jen skupiny typu `Domain Admins`, `Enterprise Admins` nebo `Administrators`, velká část reálných cest k převzetí domény zůstane úplně mimo záběr.

## Čtyři různé typy moci, které vypadají nenápadně

### 1. Schopnost získat nebo prosadit DCSync

DCSync je schopnost chovat se vůči řadiči domény jako replikační partner a vyžádat si tajemství účtů. Útočník ji nemusí mít od začátku. Někdy mu stačí jiná delegace, která mu dovolí tato oprávnění získat nebo si je vynutit přes relay.

### 2. Schopnost číst servisní tajemství

gMSA nebo synchronizační účet nemusí být interaktivní uživatel, ale může být bezpečnostně cennější než běžný admin. Jakmile lze přečíst jeho tajemství, často z něj vznikne další hop k silnější identitě.

### 3. Schopnost sahat na historické nebo skryté directory artefakty

Deleted objects, staré atributy a pomocná metadata nejsou jen forenzní vrstva. Pokud v nich zůstane heslo nebo jiná citlivá hodnota, je to stále platný vstup do prostředí.

### 4. Schopnost zneužít důvěru v jinou "technickou" identitu

Machine account, synchronizační služba nebo servisní komponenta často nevypadají jako lidský administrátor. Přesto mohou držet právě ta práva nebo tajemství, která nakonec doménovou kompromitaci rozhodnou.

## Forest: delegovaná Exchange práva, relay a DCSync

Forest je čistá ukázka toho, že mezi obyčejným servisním účtem a plným převzetím domény může stát jen několik delegovaných kroků. Po AS-REP roastu vznikne foothold v podobě `svc-alfresco`, ale ten sám o sobě ještě není doménový admin. Zásadní je až to, co BloodHound ukáže o jeho vztahu ke skupině `Exchange Windows Permissions`.

Právě tady se hodí přestat myslet v kategoriích "admin / neadmin". Účet nemá plnou kontrolu nad doménou, ale má dost silnou pozici na to, aby šlo přes PrivExchange vynutit autentizaci a relaynout ji proti LDAP:

Praktickou stránku těchto utilit jako součásti jednoho toolkitu rozebírám i v článku [Impacket](/nastroje/impacket).

```text
ntlmrelayx.py -t ldap://10.10.10.161 --escalate-user svc-alfresco
http://localhost/privexchange
```

Výsledkem není okamžitě shell. Výsledkem je změna oprávnění v adresáři, která účtu otevře DCSync. Teprve potom přichází:

```text
secretsdump.py htb.local/svc-alfresco:s3rvice@forest.htb.local
```

Forest je proto důležitý hlavně jako model: delegované provozní oprávnění k Exchange nemusí vypadat dramaticky, ale v kombinaci s relay cestou se z něj stane přímá replikační schopnost.

## Intelligence: gMSA jako most k cizí identitě

Intelligence stojí na jiném typu delegace. Účet `Tiffany.Molina` sám o sobě nepřináší interaktivní shell ani privilegovanou roli, ale dovolí přečíst `downdetector.ps1`, tedy automatizační skript, který používá `Invoke-WebRequest -UseDefaultCredentials`. Přes DNS záznam a odchyt autentizace vznikne další účet: `Ted.Graves`.

Teprve ten otevře nejcennější otázku: kdo smí číst heslo gMSA `svc_int$`?

```text
gMSADumper.py -u Ted.Graves -p Mr.Teddy -d intelligence.htb -l dc.intelligence.htb
Users or groups who can read password for svc_int$:
 > DC$
 > itsupport
```

Tady je zásadní pochopit, že gMSA není "jen servisní účet". Pokud jeho heslo smí číst nevhodná skupina nebo účet, získává útočník silnou služební identitu, která může:

- přistupovat k citlivým službám,
- žádat service ticket,
- nebo dokonce impersonovat jiného uživatele ve vhodném SPN kontextu.

Na Intelligence to vede k tomuto kroku:

```text
impacket-getST intelligence.htb/svc_int$ -spn WWW/dc.intelligence.htb -hashes :c699eaac79b69357d9dabee3379547e6 -impersonate Administrator
```

Tím se jasně ukazuje rozdíl mezi "mám servisní účet" a "mám nevýznamný účet". Servisní identita často stojí blíž centru důvěry než běžný uživatel.

## Cascade: deleted objects a historická hesla nejsou nevinný archiv

Cascade ukazuje třetí rodinu problémů. Řetězec nezačíná BloodHoundem ani relayem, ale čtením atributu `cascadeLegacyPwd`, z něhož se získá heslo `r.thompson`. Následně přibývají další stopy ve sdílených souborech a auditních nástrojích, až se účet `ArkSvc` dostane k deleted objects.

Právě tam se objeví historický `TempAdmin`:

```powershell
Get-ADObject -filter 'isdeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property *
```

spolu s dalším `cascadeLegacyPwd`, které po dekódování vydá heslo administrátora.

Cascade je důležitá připomínka, že AD není jen aktuální stav objektů. Pokud v deleted objects, poznámkách nebo historických atributech přežívá skutečné tajemství, je to pořád aktivní bezpečnostní problém. Účet může být smazaný, ale jeho heslo nebo pomocný atribut mohou dál otevřít cestu k jiné identitě.

## APT: hodnotná nemusí být jen lidská identita

APT přidává důležitý kontrast. Hlavní řetězec začíná únikem `backup.zip`, z nějž se offline vytěží doménová tajemství a otevře se účet `henry.vinson_adm`. Pro doménovou kompromitaci je ale důležité něco jiného: vynucená autentizace účtu stroje `APT$`.

```text
[SMB] NTLMv1 Username : HTB\APT$
```

Po cracknutí odpovědi a následném dumpu doménových tajemství je zřejmé, že strojová identita může být pro útok stejně cenná jako uživatelská. Obranně je to zásadní, protože machine accounty bývají podceňované:

- často se neberou jako "skutečné účty",
- méně se kolem nich sleduje reuse a offline kompromitace,
- ale v konkrétní konfiguraci mohou mít přístup, který stačí k dalšímu replikačnímu kroku.

APT tak dobře doplňuje zbytek článku: nepřímá cesta k doménovým tajemstvím nemusí vést přes helpdesk admina nebo servisního uživatele. Může vést i přes stroj.

## Monteverde: hraniční, ale důležitý případ synchronizační služby

Monteverde není čistý příklad ACL delegace, relaye ani deleted objects. Přesto do stejné rodiny patří. Účet `mhope` otevře host s Azure AD Connect a ten v sobě drží dešifrovatelné synchronizační tajemství. Z databáze `ADSync` a přítomné `mcrypt.dll` pak lze dostat doménové přihlašovací údaje pro `administrator`.

Tenhle případ je důležitý hlavně jako varování před příliš úzkou definicí delegovaných práv. Někdy nejde o ACL v AD objektu. Jde o to, že vedle domény žije synchronizační komponenta, která v sobě drží plnohodnotné identity tajemství:

- běží s vysokou důvěrou,
- musí umět číst a zapisovat do identitního systému,
- a tím se sama stává cílem stejné váhy jako doménový kontroler.

Monteverde proto funguje jako rozšíření hlavního modelu: privilegovaný přístup v identitním ekosystému nemusí být vidět jako skupina nebo právo v BloodHoundu. Může být ukrytý v pomocné službě, která synchronizuje identity jinam.

## Jak podobné cesty hledat systematicky

Při analýze AD prostředí je užitečné jít po typech schopností, ne po jménech skupin.

### Kdo může měnit nebo získat replikační oprávnění

- ACL na doméně a citlivých objektech,
- delegace spojené s Exchange a jinými provozními komponentami,
- cesty, které relay přes LDAP promění v DCSync.

### Kdo může číst heslo jiného účtu

- gMSA,
- synchronizační tajemství,
- databáze a konfigurace pomocných služeb,
- machine account workflow a zálohy.

### Kde přežívají historická tajemství

- deleted objects,
- komentáře a poznámky účtů,
- staré atributy typu `legacyPwd`,
- exporty a auditní databáze.

### Které technické identity mají větší dopad, než se zdá

- servisní účty,
- machine accounty,
- synchronizační služby,
- maintenance a monitoring komponenty s přístupem do AD.

## Obrana a hardening

### 1. Auditovat ACL a delegace, ne jen skupiny

Členství v `Domain Admins` je jen jedna z mnoha cest. Je potřeba pravidelně reviewovat i delegovaná oprávnění, která mohou přes relay, Exchange nebo jiné workflow vést ke změně adresářových práv.

### 2. gMSA a servisní tajemství brát jako kritický majetek

Otázka "kdo může číst heslo gMSA" je stejně důležitá jako "kdo je lokální admin na serveru". Jakmile je odpověď příliš široká, vzniká silný laterální most.

### 3. Čistit historická a pomocná tajemství

Deleted objects, `legacyPwd`, auditní DB a staré konfigurace nesmějí obsahovat reálně použitelné heslo. Historie v AD nebo ve sdíleném souboru je pořád součást útokové plochy.

### 4. Nepodceňovat technické identity

Machine account nebo sync služba nejsou "jen systémové účty". Pokud drží přístup k replikaci, synchronizaci nebo autentizaci jménem jiných účtů, mají prakticky stejnou váhu jako lidský administrátor.

### 5. Sledovat pomocné identity systémy kolem AD

Azure AD Connect, Exchange a další přilehlé komponenty je potřeba reviewovat jako součást identity hranice. Ne jako vedlejší infrastrukturu.

## Shrnutí klíčových poznatků

- Doménová kompromitace často vzniká z delegovaných práv a servisních tajemství, ne z přímého členství v administrátorské skupině.
- Forest ukazuje cestu od provozní delegace přes relay k DCSync.
- Intelligence ukazuje, jak nebezpečné je právo číst heslo gMSA.
- Cascade připomíná, že deleted objects a historické atributy mohou dál obsahovat aktivní tajemství.
- APT a Monteverde rozšiřují model o strojové identity a synchronizační služby, které bývají podceňované, ale dopad mají plně doménový.

## Co si odnést do praxe

- Při AD review se neptej jen "kdo je admin". Ptej se i "kdo umí měnit práva", "kdo umí číst cizí tajemství" a "kde zůstala historická hesla".
- Servisní a technické identity bývají v praxi často silnější, než se zdá z názvu účtu. gMSA, sync účet nebo machine account mohou být bezpečnostně stejně cenné jako lidský administrátor.
- Jakmile je možné přejít z nenápadné delegace na DCSync, impersonaci nebo čtení synchronizačního tajemství, je rozdíl proti přímému `Domain Admin` spíš kosmetický než reálný.
