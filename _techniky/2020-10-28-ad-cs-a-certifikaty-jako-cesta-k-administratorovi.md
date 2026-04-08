---
layout: post
author: Tomáš Matějíček
title: "AD CS a certifikáty jako cesta k Administratorovi"
date: 2020-10-28
tags: adcs active-directory windows kerberos certificates pkinit
---
## Úvod a kontext

Když se v Active Directory mluví o kompromitaci identity, většina lidí si představí heslo, NT hash nebo Kerberos ticket. Active Directory Certificate Services ale otevírají jinou rovinu. Pokud doména důvěřuje certifikátu pro autentizaci, může být certifikát pro privilegovaný účet stejně cenný jako jeho heslo, a někdy i praktičtější. Útočník pak nepotřebuje znát password. Stačí mu schopnost získat certifikát, který se na doméně mapuje na cílovou identitu.

Na Anubis to bylo vidět velmi čistě. První foothold i laterální pohyb jsou důležité jen proto, aby se útočník dostal k prostředí, kde lze pracovat se šablonami certifikátů. Skutečný zlom přijde ve chvíli, kdy se upraví šablona `Web`, vystaví certifikát pro `Administrator` a přes PKINIT se z něj získá TGT. Od této chvíle už nejde o heslo nebo hash. Jde o plnohodnotný alternativní autentizační materiál.

## Co AD CS v doméně skutečně znamená

AD CS je doménová certifikační služba. Vydává certifikáty, které se pak používají pro různé účely:

- TLS a webové služby,
- šifrování a podepisování,
- smart card logon nebo jiné formy doménové autentizace,
- interní servisní komunikaci.

Z bezpečnostního pohledu je důležité, že certifikát není jen "soubor s klíčem". Je to identitní artefakt, kterému doména může věřit stejně silně jako Kerberos přihlášení. Jakmile lze vystavit certifikát pro privilegovanou identitu nebo změnit šablonu tak, aby to umožnila, vzniká přímá cesta k jejímu účtu.

## Základní pojmy, které je potřeba neplést

### Certifikační autorita

CA je služba, která žádost podepíše a vydá certifikát. Samotná existence CA ještě není problém. Problém vzniká v tom, kdo smí požádat o jaký certifikát a podle jakých pravidel.

### Šablona certifikátu

Šablona definuje, jaký typ certifikátu lze vydat. Prakticky určuje například:

- kdo smí o certifikát žádat,
- jaké subject nebo SAN hodnoty lze použít,
- jaké EKU se do certifikátu dostanou,
- jestli jde certifikát použít pro klientskou autentizaci.

Právě šablona je často skutečnou hranicí důvěry.

### Enroll právo

To určuje, kdo může podle dané šablony certifikát získat. Pokud má šablona příliš široké `Enroll` nebo `Autoenroll`, problém nezačíná až u kompromitace CA. Začíná už u samotné dostupnosti šablony.

### EKU a aplikační politika

Extended Key Usage a související aplikační politika říkají, k čemu lze certifikát použít. Z pohledu AD útoku je zásadní hlavně to, zda certifikát dovoluje klientskou autentizaci nebo smart card logon. Pokud ne, může být vhodný pro TLS, ale ne pro přihlášení do domény. Pokud ano, z obyčejného certifikátu se stává autentizační materiál.

### Mapování certifikátu na identitu

Doména musí nějak rozhodnout, komu certifikát patří. Často se to děje přes UPN nebo jiný identitní údaj v subjectu či SAN. Právě tady vzniká kritický moment: pokud lze vystavit certifikát, jehož identita ukazuje na `Administrator`, vzniká prakticky přímé převzetí této identity.

## Kde se z běžné PKI služby stává útočná plocha

AD CS není nebezpečné proto, že "vydává certifikáty". Nebezpečné je tehdy, když se sejdou tyto podmínky:

- útočník může měnit šablonu nebo její bezpečnostní vlastnosti,
- může žádat o certifikát podle šablony, která dovoluje autentizaci,
- může ovlivnit identitu, pro kterou se certifikát vystaví,
- a doména pak tento certifikát přijme jako legitimní přihlášení.

To je důležitý rozdíl proti heslům a hashům. U password compromise útočník obvykle získá existující tajemství. U AD CS často vytvoří nový, plně legitimní identitní artefakt.

## Anubis: od šablony `Web` k PKINIT jako `Administrator`

Na Anubis se cesta k AD CS neotevírá sama od sebe. Nejprve je potřeba dostat se do interního prostředí, laterálně se posunout na host `earth` a ověřit, že v doméně skutečně běží certifikační autorita. Share `CertEnroll` a vydané CA artefakty to potvrdí velmi rychle:

```text
\\earth\CertEnroll
earth.windcorp.htb_windcorp-CA.crt
windcorp-CA.crl
```

To ale ještě není exploitační moment. Ten přichází až s pochopením šablony `Web`. V daném prostředí šlo upravit její EKU tak, aby se z původně méně nebezpečné šablony stal nástroj pro autentizaci:

```powershell
$EKUs=@("1.3.6.1.5.5.7.3.2", "1.3.6.1.4.1.311.20.2.2")
Set-ADObject "CN=Web,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=windcorp,DC=htb" -Add @{pKIExtendedKeyUsage=$EKUs;"msPKI-Certificate-Application-Policy"=$EKUs}
```

Tím se stalo něco velmi podstatného: šablona začala vydávat certifikát, který už není jen "na web". Je použitelný pro klientskou autentizaci v doméně.

Další krok je žádost o certifikát pro privilegovanou identitu. Zde je kritické právě mapování identity. V žádosti se objeví UPN nebo jiný subject odpovídající `administrator@windcorp.htb`. Praktickou práci s tímto typem materiálu rozebírám i v článku [OpenSSL](/nastroje/openssl):

```text
openssl req -config admin.cnf -subj "/DC=htb/DC=windcorp/CN=Users/CN=Administrator" -new -nodes -sha256 -out admin.req -keyout admin.key
certreq.exe -submit -config earth.windcorp.htb\windcorp-CA -attrib "CertificateTemplate:Web" admin.req admin.cer
```

Jakmile CA takový certifikát vydá, nevznikne jen další soubor. Vznikne důvěryhodná identita pro Kerberos.

## Co přesně dělá PKINIT

PKINIT je způsob, jak použít certifikát a soukromý klíč při Kerberos autentizaci. Prakticky to znamená, že místo klasického hesla nebo hashe lze požádat o TGT s využitím certifikátu:

```text
kinit -X X509_user_identity=FILE:admin.cer,admin.key Administrator@WINDCORP.HTB
```

Pokud doména certifikátu věří a namapuje ho na `Administrator`, výsledek je z pohledu zbytku prostředí stejný, jako kdyby se `Administrator` právě autentizoval sám. Proto je AD CS tak silné. Nejde o vedlejší feature. Jde o alternativní vstup do identity.

Na Anubis pak už logicky následuje WinRM:

Prakticky k tomu, co od takového shellu čekat a kdy je WinRM nejlepší další krok, viz také [Evil-WinRM](/nastroje/evil-winrm).

```text
evil-winrm -i earth.windcorp.htb -u administrator -r WINDCORP.HTB
```

a tím i plný administrátorský přístup.

## V čem se certifikát liší od hesla nebo hashe

Z obranného pohledu je užitečné si explicitně říct, proč je AD CS chyba jiný problém než běžný credential leak.

### Útočník nemusí znát původní heslo

Doména neověřuje password. Ověřuje certifikát a jeho vazbu na identitu.

### Vzniká nový autentizační materiál

Nejde jen o znovupoužití uniklého tajemství. Útočník si často vytvoří vlastní certifikát, který vypadá legitimně a zapadá do běžného PKI provozu.

### Cleanup bývá složitější

U změny hesla je reakce zřejmá. U certifikátů je potřeba myslet i na:

- revokaci,
- platnost šablon,
- audit vydaných certifikátů,
- a odstranění chybných práv na šabloně.

### Audit jde často po špatné vrstvě

Týmy bývají zvyklé auditovat:

- `Domain Admins`,
- hesla servisních účtů,
- NTLM relay,
- DCSync práva.

AD CS přitom někdy zůstává mimo hlavní pozornost, i když praktický dopad může být stejný.

## Na co se při analýze zaměřit

Při review AD CS nedává smysl začínat obecnou otázkou "je tu CA?". Smysluplnější jsou konkrétní kontrolní body.

### Kdo smí měnit šablony

- ACL na objektech šablon,
- členství ve skupinách spojených s certifikační správou,
- delegovaná práva v konfigurační partition.

### Kdo smí podle šablony žádat

- `Enroll`,
- `Autoenroll`,
- a reálný okruh účtů, které tato práva drží.

### Co šablona dovolí vystavit

- klientskou autentizaci,
- smart card logon,
- volitelný subject nebo SAN,
- UPN jiného uživatele,
- nebo jiné údaje, které se mapují na privilegovanou identitu.

### Jak se vydané certifikáty používají

- PKINIT,
- smart card logon,
- interní API nebo management kanály,
- vzdálené přihlášení přes WinRM nebo podobné služby.

## Obrana a hardening

### 1. Auditovat šablony jako privilegované objekty

Šablona, která umí vydat autentizační certifikát, je bezpečnostně srovnatelná s privilegiovanou identity cestou. ACL na ní musí být stejně přísné jako na jiných kritických AD objektech.

### 2. Omezit, kdo může žádat a co lze zapsat do identity

Šablona nemá umožňovat libovolné mapování na jinou identitu, pokud k tomu není velmi silný provozní důvod. Totéž platí pro široce rozdané `Enroll` právo.

### 3. Přísně hlídat EKU a aplikační politiku

Přidání klientské autentizace nebo smart card logon do nevhodné šablony je přesně ten druh změny, který mění PKI z pomocné služby na přímou cestu k převzetí identity.

### 4. Monitorovat issuance pro privilegované identity

Podezřelé je zejména:

- vystavení certifikátu pro administrátorský UPN,
- neobvyklé použití autentizačních šablon,
- změna EKU na existující šabloně,
- žádosti z nečekaných hostů nebo účtů.

### 5. Držet AD CS ve stejné bezpečnostní kategorii jako jiné identity služby

CA, šablony a issuance workflow nesmí být "někde bokem". Pokud doména přijímá certifikát jako identitu, pak chyba v AD CS je identity compromise.

## Shrnutí klíčových poznatků

- AD CS je alternativní vstup do identity v Active Directory, ne jen podpůrná PKI služba.
- Kritická chyba vzniká tehdy, když útočník může změnit šablonu nebo získat certifikát, který se mapuje na privilegovaný účet.
- Anubis ukazuje přesný řetězec: úprava šablony `Web`, vystavení certifikátu pro `Administrator` a následný PKINIT.
- Obrana stojí hlavně na ACL šablon, restrikci `Enroll` práv, kontrole EKU a auditu issuance pro privilegované identity.

## Co si odnést do praxe

- Při AD auditu se neptej jen na hesla, relay a skupiny. Ptej se i na to, kdo může měnit šablony a kdo může získat autentizační certifikát.
- Certifikát pro `Administrator` je z praktického pohledu stejná katastrofa jako znalost jeho hesla, někdy i horší, protože nevypadá jako běžný password compromise.
- Jakmile certifikáty slouží k přihlášení do domény, musí být AD CS spravované jako kritická identity vrstva, ne jako vedlejší PKI infrastruktura.
