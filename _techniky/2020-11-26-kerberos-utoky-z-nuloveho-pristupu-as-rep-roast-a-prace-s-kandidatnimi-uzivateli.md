---
layout: post
author: Tomáš Matějíček
title: "Kerberos útoky z nulového přístupu: AS-REP roast a práce s kandidátními uživateli"
date: 2020-11-26
tags: kerberos active-directory windows asrep-roast enumeration
---
## Úvod a kontext

Když se řekne útok na Active Directory, mnoho lidí si představí až fázi po footholdu: BloodHound, relay, DCSync nebo dump hashů. Velká část doménových řetězců ale začíná mnohem dřív. Stačí veřejný web, dokumenty, jména zaměstnanců a několik odpovědí od Kerberosu. Právě tomu dává smysl říkat nulový přístup: útočník ještě nemá platné přihlašovací údaje, ale už umí z domény vytěžit strukturu účtů a někdy i crackovatelný materiál.

Typickým příkladem je AS-REP roast. Ten je užitečný právě proto, že nestojí na exploitaci služby ani na hrubé síle proti login formuláři. Vychází z toho, že některý účet nemá zapnutou Kerberos pre-auth, takže řadič domény vrátí odpověď, kterou lze lámat offline. Jenže samotný roast je až druhý krok. Nejdřív je potřeba mít sadu realistických kandidátních uživatelů a umět oddělit validaci jmen, roasting a testování hesel.

## Co přesně znamená nulový přístup

Nulový přístup neznamená, že útočník neví vůbec nic. Znamená jen to, že zatím nedrží platný účet. I bez něj ale často může pracovat s:

- veřejnými jmény zaměstnanců,
- e-mailovými adresami,
- metadaty dokumentů,
- naming konvencí firmy,
- a rozdíly v odpovědích Kerberosu nebo LDAPu.

To je důležité i metodicky. Kerberos útok z nulového přístupu obvykle nezačíná payloadem. Začíná otázkou, jak vypadá jmenný prostor cílové organizace a které účty vůbec stojí za další test.

## Proč jsou kandidátní uživatelé důležitější než samotný tool

AS-REP roast nedává smysl proti náhodnému seznamu jmen. Dává smysl teprve tehdy, když kandidátní účty odpovídají tomu, jak organizace skutečně tvoří identity. V praxi se vyplatí hledat hlavně tři zdroje.

### 1. Veřejná jména a role

Na veřejném webu se často objevují seznamy zaměstnanců, vedení, kontaktních osob nebo autorů článků. Na Sauně právě tahle data stačila k tomu, aby vznikl realistický seznam jmen a z něj kandidátní účty typu `FSmith`.

### 2. Metadata dokumentů

Na Intelligence byly rozhodující PDF dokumenty. Samotný obsah nebyl tak důležitý jako jejich metadata: `Creator`, autor a datumové vzory v názvech souborů. Díky tomu šlo vygenerovat uživatele jako `William.Lee`, `Jose.Williams` nebo `Tiffany.Molina` a dál s nimi pracovat proti doméně.

### 3. Samotná doména bez přihlášení

Někdy prozradí dost i samotná infrastruktura. Forest dovolil přes anonymní dotaz vytáhnout širší seznam AD účtů a teprve nad ním dávalo smysl spouštět `GetNPUsers.py`. V takové situaci je chyba rovnou přejít k lámání hesel. Nejdřív je potřeba zjistit, které účty skutečně existují a které z nich jsou zajímavé z pohledu Kerberosu.

## Tři různé kroky, které se často pletou dohromady

V praxi se vyplatí přísně oddělit tři činnosti, protože každá z nich znamená něco jiného a vyžaduje jinou obranu.

### Validace uživatelských jmen

Tady ještě nejde o heslo. Cílem je zjistit, které identity v doméně vůbec existují. Typicky pomůže `kerbrute userenum` nebo jiná technika, která rozliší platné a neplatné jméno.

Praktickou stránku tohoto úzkého kroku rozebírám samostatně i v článku [Kerbrute](/nastroje/kerbrute).

```bash
./kerbrute_linux_amd64 userenum --dc intelligence.htb -d intelligence.htb users.txt
```

To je důležité hlavně proto, že bez validních uživatelů bude další práce šum. Obranně je to zároveň jiný problém než password spraying: útočník zatím nezkouší hesla, jen mapuje identitní prostor.

### AS-REP roast

AS-REP roast už stojí na validních jménech, ale pořád ještě nezkouší heslo online. Zkoumá, zda některý účet dovolí vydat Kerberos odpověď bez pre-auth. Pokud ano, vznikne materiál pro offline cracking.

```bash
GetNPUsers.py -format hashcat -usersfile users.txt -outputfile hashes.asreproast domain.local/
```

Tahle fáze je pro útočníka levná a pro obránce nepříjemná. Jakmile hash opustí řadič domény, síťová kontrola už končí a zbytek se láme offline. Praktickou logiku této vrstvy rozebírám i v článku [Hashcat](/nastroje/hashcat).

### Testování hesel a reuse

Teprve třetí vrstva je práce s heslem. Někdy přijde po cracknutí AS-REP hashe, jindy z dokumentu nebo z výchozího hesla v interním oběžníku. Na Intelligence byl právě tenhle kontrast dobře vidět: metadata pomohla najít uživatele, ale foothold nakonec nevznikl přes roast, nýbrž přes defaultní heslo `NewIntelligenceCorpUser9876` pro účet `Tiffany.Molina`.

Právě proto je dobré mluvit spíš o nulovém přístupu ke Kerberosu než jen o AS-REP roastu. Roast je jen jedna z cest, co s kandidátními účty dál dělat.

## Jak tenhle vzorec vypadal v praxi

### Forest: anonymní enumerace a účet bez pre-auth

Forest je velmi čistý příklad toho, kdy sama doména vydá seznam účtů a jeden z nich je přímo roastovatelný. `GetADUsers.py` poskytl dostatečný přehled identit a `GetNPUsers.py` pak ukázal, že `svc-alfresco` nemá Kerberos pre-auth. Výstup šel přímo do `hashcat` a po cracknutí hesla `s3rvice` už byl k dispozici stabilní vstup přes WinRM. Praktickou stránku těchto utilit rozebírám podrobněji v článku [Impacket pro AD enumeraci a první identity](/nastroje/impacket-pro-ad-enumeraci-a-prvni-identity).

Na tomhle případu je důležitá jedna věc: foothold nevznikl kvůli slabému webu nebo SMB share. Vznikl čistě z kombinace identitní enumerace a špatně nastaveného servisního účtu.

### Sauna: veřejná jména, naming konvence a až potom roast

Sauna ukazuje jinou výchozí situaci. Doména sama předem nevydá seznam účtů, ale veřejný web banky prozradí jména zaměstnanců. Z nich jde odvodit naming pattern a připravit sadu kandidátů. Teprve potom dává smysl spustit `GetNPUsers.py` a hledat účet bez pre-auth.

Právě tady je dobře vidět, že nejcennější část práce nebyl samotný nástroj, ale příprava kvalitního seznamu jmen. Účet `FSmith` se neobjevil náhodou. Objevil se proto, že kandidátní uživatelé odpovídali tomu, jak cílová organizace tvoří identity.

### Intelligence: kandidátní uživatelé nejsou jen pro roast

Intelligence je dobrý protiváhový příklad. Veřejné dokumenty a jejich metadata pomohly sestavit validní uživatele a potvrdit je proti doméně. Jenže foothold nakonec nevznikl přes účet bez pre-auth. Rozhodující bylo, že jeden historický dokument obsahoval výchozí heslo, které fungovalo pro `Tiffany.Molina`.

To je velmi důležitá lekce pro praxi. Jakmile už jednou máte kvalitní sadu kandidátních účtů, nevede z ní jen cesta k AS-REP roastu. Může vést i k opatrnému testu výchozích hesel, k ověřování reuse nebo k bezpečnějšímu password sprayingu s úzkým rozsahem.

## Jak postupovat po úspěšném roastu

Po cracknutí AS-REP hashe vzniká pokušení skočit rovnou na další velký AD útok. Prakticky je ale lepší držet jednoduchou disciplínu.

### 1. Ověřit, kde účet skutečně funguje

Heslo z roastu ještě neříká, jestli účet dovolí:

- WinRM,
- SMB přístup,
- LDAP bind,
- nebo jen běžné doménové přihlášení.

Na Forestu dával hned smysl WinRM, protože port `5985` byl otevřený a účet `svc-alfresco` byl provozně použitelný. Na jiném hostu může být první bezpečný krok jen čtení share nebo LDAP enumerace.

Praktickou stránku práce s WinRM shellem rozebírám samostatně v článku [Evil-WinRM](/nastroje/evil-winrm).

### 2. Odhadnout, jaký typ účtu jste získali

Servisní účet bez pre-auth bývá cennější než běžný uživatel, ale ne nutně interaktivní. Naopak lidský účet může rychle otevřít WinRM nebo přístup do interních share. Rozdíl mezi `svc-alfresco`, `FSmith` a `Tiffany.Molina` dobře ukazuje, že hodnota účtu neleží jen v hesle, ale i v jeho roli v doméně.

### 3. Držet scope prvního kroku úzký

Nulový přístup je levný právě proto, že je opatrný. Jakmile po prvním úspěchu přejdete do hlučného password sprayingu nebo agresivní enumerace, zvedáte cenu útoku i šanci na detekci. Prakticky se vyplatí nejdřív potvrdit, že nově získaný účet opravdu otevírá další vrstvu domény.

## Obrana a bezpečnější návrh

### 1. Nepouštět servisní účty bez pre-auth

Účty bez Kerberos pre-auth mají být výjimečná a dobře zdůvodněná odchylka. Pokud neexistuje opravdu silný provozní důvod, je to zbytečný zdroj offline crackovatelného materiálu.

### 2. Pracovat s tím, že veřejná jména jsou součást útokové plochy

Seznam zaměstnanců není zranitelnost sám o sobě, ale zlevňuje první fázi útoku. Obrana proto nestojí na tom jména skrývat, ale na tom, že doména nesmí reagovat rozdílně a účty nesmí padat na slabých heslech nebo vypnuté pre-auth.

### 3. Auditovat výchozí a historická hesla v dokumentech

Intelligence dobře ukazuje, že identitní útok často nevznikne kvůli Kerberosu samotnému, ale kvůli interní disciplíně. Historický dokument s výchozím heslem může mít stejnou hodnotu jako webový exploit.

### 4. Detekovat opakující se práci s kandidátními účty

Z obranného pohledu je podezřelý nejen password spray, ale i systematická validace jmen a opakované Kerberos požadavky nad širším seznamem identit. Útočník si tak často připravuje půdu ještě před prvním platným loginem.

### 5. Omezit reuse a přehnaně silné servisní identity

I když je první účet získaný "jen" přes AS-REP roast, skutečný dopad pak obvykle rozhodne až reuse hesla, WinRM přístup nebo přehnaná oprávnění účtu v doméně. Obrana proto nekončí u Kerberosu, ale pokračuje do segmentace rolí a správy servisních účtů.

## Shrnutí klíčových poznatků

- Nulový přístup k Active Directory často začíná prací s kandidátními uživateli, ne exploitem služby.
- AS-REP roast je jen jedna část širšího workflow: nejdřív je potřeba účty najít a validovat, teprve potom má smysl hledat ty bez pre-auth.
- Forest, Sauna a Intelligence ukazují tři různé varianty stejného problému: anonymní enumeraci, veřejná jména a metadata dokumentů.
- Rozdíl mezi user enumeration, AS-REP roastem a testováním hesel je zásadní jak pro útok, tak pro obranu.

## Co si odnést do praxe

- Když na doméně ještě nemáte žádný účet, neznamená to, že nemáte žádnou útokovou plochu. Veřejná jména, metadata a odpovědi Kerberosu často stačí k prvnímu reálnému kroku.
- Při obranném auditu má smysl ptát se ve správném pořadí: které účty lze odvodit z veřejných informací, které z nich jsou validní a které z nich dovolují AS-REP roast nebo reuse výchozího hesla.
- Účet bez pre-auth není jen "slabší Kerberos". Je to přímo offline crack primitivum, které obchází většinu online ochranných mechanismů.
