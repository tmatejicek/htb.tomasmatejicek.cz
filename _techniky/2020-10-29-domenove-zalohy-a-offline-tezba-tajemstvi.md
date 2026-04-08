---
layout: post
author: Tomáš Matějíček
title: "Doménové zálohy a offline těžba tajemství"
date: 2020-10-29
tags: active-directory backup ntds windows dcsync secrets
---
## Úvod a kontext

Únik doménové zálohy patří k těm incidentům, které se snadno podcení. Administrátor často vnímá `backup.zip` jako kopii pro recovery a ne jako aktivní identitní riziko. Jenže pokud záloha obsahuje:

- `ntds.dit`,
- `SYSTEM`,
- export registru,
- nebo jiná tajemství účtů a služeb,

pak už nejde o "soubor se zálohou". Jde o offline kopii identity vrstvy celé domény.

APT je velmi dobrý příklad právě proto, že ukazuje dvě věci najednou. Nejdřív samotná doménová záloha vydá dost materiálu na první účty a uložená tajemství. A potom se celý řetězec uzavře přes strojový účet `APT$` a DCSync. Tím je dobře vidět, že hodnota zálohy neleží jen v heslech uživatelů. Leží v celé mapě identit, hashů a vedlejších tajemství, která se z ní dají vytěžit offline.

## Co z doménové zálohy typicky uniká

Z bezpečnostního pohledu není důležité jen to, že v archivu leží nějaké "AD soubory". Důležité je, jaký typ schopností útočník po rozbalení získá.

### `ntds.dit`

Databáze Active Directory, která obsahuje identity a jejich tajemství v podobě, z níž lze offline vytěžit NTLM hash nebo další autentizační materiál.

### `SYSTEM`

Samotné `ntds.dit` nestačí vždy k použitelnému offline zpracování. K rekonstrukci potřebných klíčů a tajemství bývá potřeba i registry hive `SYSTEM`.

### Strojové účty a servisní identity

Ve výpisu nejsou jen lidští uživatelé. Často tam bývají:

- machine accounty,
- servisní účty,
- historické administrativní identity.

Právě ty bývají z obranného pohledu nejnebezpečnější, protože často otevírají jiné technické cesty než běžné heslo uživatele.

### Vedlejší uložená tajemství

Jakmile první získaný účet dovolí číst registry nebo konfigurace aplikací, přidají se další hesla, která v původní záloze ani nemusela být přímo vidět.

## APT: od `backup.zip` k plnému doménovému kompromisu

Na APT je první důležitá lekce už v tom, že záloha neleží za silnou bariérou. Přechod na IPv6 odhalí kompletní AD infrastrukturu a anonymní SMB share `backup`, v němž je `backup.zip`.

To je dobré připomenutí samo o sobě. Z pohledu útoku není zásadní, jestli se k záloze došlo přes exploit, nebo přes špatně vystavený share. Jakmile je archiv dostupný, zbytek už může proběhnout zcela offline.

Po získání hesla k ZIPu a rozbalení se ukáže přesně ta kombinace, která je pro AD kritická:

- `ntds.dit`,
- `SYSTEM`.

To už dovolí spustit offline extrakci tajemství:

```bash
impacket-secretsdump -ntds "Active Directory/ntds.dit" -system registry/SYSTEM -hashes lmhash:nthash LOCAL -outputfile ntlm-extract
```

Tady je důležité správně chápat dopad. Útočník nepotřebuje dál komunikovat s doménou, aby z těchto souborů získal identitní materiál. První těžba probíhá mimo dohled cílového prostředí.

## Proč je offline těžba tak silná

Jakmile se tajemství dostanou mimo doménu, obránce přichází o velkou část tradičních kontrol:

- žádný lockout policy,
- žádný rate limiting na login endpointu,
- žádná síťová detekce proti dalším pokusům uvnitř samotné těžby,
- a často ani žádný okamžitý signál, že se se soubory pracuje.

To je zásadní rozdíl proti běžnému password sprayingu nebo online útoku. Hodnota uniklé zálohy spočívá právě v tom, že z identity útoku se stane offline analytická práce.

## APT: hash není konec, ale mezikrok

Na APT je velmi užitečné, že první offline těžba nevede rovnou k `Administrator`. Místo toho otevře účet `henry.vinson` a přes registry i uložené údaje aplikace `GiganticHostingManagementSystem`, které dají `henry.vinson_adm`.

To je důležitá lekce pro praxi. Doménová záloha často nedává jen "finální hash admina". Častěji otevře několik mezikroků:

- běžný nebo servisní účet,
- přístup k registru,
- čitelné aplikační tajemství,
- další management účet.

Právě tak se z offline dat stane reálný online foothold.

## Strojové účty nejsou vedlejší detail

Na APT pak přichází druhá velmi důležitá část: účet stroje `APT$`. Přes vynucenou autentizaci a odchyt NTLMv1 response:

```text
[SMB] NTLMv1 Username : HTB\APT$
```

vznikne další použitelný materiál, který po cracknutí otevře cestu k:

```bash
impacket-secretsdump 'apt.htb/APT$@apt.htb.local'
```

a nakonec k hashi `Administrator`.

Pro článek o zálohách je to důležité právě proto, že ukazuje plný dopad technických identit. Záloha a z ní vytěžené účty nevedou jen k běžnému loginu. Mohou otevřít i strojový nebo servisní kontext, který je pro replikační útok ještě cennější.

## Jak podobný incident číst správně

Únik doménové zálohy je užitečné číst ve čtyřech vrstvách.

### 1. Co přesně uniklo

- jen dokumenty,
- nebo přímo `ntds.dit`, `SYSTEM`, registry exporty, konfigurace a další identity artefakty.

### 2. Co lze zpracovat offline

- NTLM hashe,
- strojové účty,
- historické identity,
- další informace o topologii domény.

### 3. Co z prvního účtu otevře další tajemství

- registry aplikací,
- management účty,
- uložená hesla,
- laterální přístup na jiné hosty.

### 4. Který technický účet uzavírá řetězec

Na APT je to `APT$`. Na jiném prostředí to může být sync účet, servisní identita nebo jiný technický kontext s vyšší váhou než běžný uživatel.

## Rozdíl proti článku o delegovaných právech v AD

Tady je důležité držet scope odděleně od už napsaného článku o delegovaných právech.

- Článek o delegovaných právech řeší, kdo v doméně už má sílu číst gMSA, změnit ACL nebo získat DCSync.
- Tento článek řeší, co se stane, když unikne samotná offline kopie identity vrstvy a další kroky se rozběhnou z ní.

Jinými slovy: tam je středem delegace uvnitř živé domény. Tady je středem kompromitovaná záloha jako vstupní artefakt.

## Obrana a hardening

### 1. Chránit zálohy jako produkční identitní data

`backup.zip` s `ntds.dit` a `SYSTEM` není obyčejný archiv. Je to prakticky export doménových tajemství. Musí mít stejnou ochranu jako řadič domény sám.

### 2. Nezveřejňovat zálohy přes dostupné share nebo pomocné exporty

Anonymní nebo slabě chráněný share se zálohami je z pohledu útočníka ekvivalent veřejně vystavené identity infrastruktury.

### 3. Omezit vedlejší úniky po prvním footholdu

První účet získaný ze zálohy často otevře další registry a aplikační tajemství. Proto nestačí řešit jen samotné AD soubory. Je potřeba minimalizovat i ukládání pomocných hesel v aplikacích a profilech.

### 4. Zakázat NTLMv1 a auditovat machine accounty

APT dobře ukazuje, že i po úniku zálohy může finální doménový kompromis rozhodnout až technický účet a starý autentizační mechanismus. NTLMv1 a podceněné machine accounty zvyšují dopad každého takového incidentu.

### 5. Mít plán reakce na kompromitovanou zálohu

Pokud jednou unikla offline kopie AD tajemství, běžná reakce "změníme jedno heslo" nestačí. Je potřeba počítat s širší rotací a s tím, že útočník mohl část materiálu zpracovat bez jakéhokoli síťového signálu.

## Shrnutí klíčových poznatků

- Doménová záloha s `ntds.dit` a `SYSTEM` není běžný recovery artefakt, ale offline kopie identity vrstvy domény.
- APT ukazuje celý řetězec: veřejně dostupný `backup.zip`, offline těžba hashů, první management účet a nakonec technický účet `APT$`, který uzavře cestu k `Administrator`.
- Největší síla takového úniku spočívá v tom, že velká část práce proběhne offline mimo dohled živé domény.
- Obrana stojí hlavně na tom, že zálohy jsou chráněné jako nejcitlivější identitní data a že technické účty a staré autentizační mechanizmy nezvyšují dopad ještě dál.

## Co si odnést do praxe

- Když unikne doménová záloha, neřeš to jako ztracený archiv. Řeš to jako kompromitaci identity infrastruktury.
- Offline vytěžený hash je často jen první krok. Skutečný dopad vzniká až ve chvíli, kdy z něj otevřeš další management účet, registry nebo technickou identitu.
- Machine accounty, NTLMv1 a vedlejší uložená tajemství dělají z kompromitované zálohy ještě horší incident, než by napovídal samotný `ntds.dit`.
