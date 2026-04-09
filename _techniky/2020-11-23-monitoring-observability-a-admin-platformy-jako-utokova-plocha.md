---
layout: post
author: Tomáš Matějíček
title: "Monitoring, observability a admin platformy jako útoková plocha"
date: 2020-11-23
tags: monitoring observability admin-platforms splunk kibana centreon
---
## Úvod a kontext

Monitoring a observability systémy se často vnímají jako pasivní vrstva. Dashboard ukazuje stav služeb, sbírají se logy, někde běží agent a tím to končí. V praxi je to naopak jedna z nejcitlivějších částí prostředí. Tyto platformy mívají přístup:

- k tajemstvím,
- k dalším serverům,
- k logům z celé infrastruktury,
- a často i k mechanismům, které umí spouštět kontroly, pluginy nebo vzdálené akce.

Doctor, Haystack, Wall a Omni ukazují čtyři různé podoby stejného problému. Splunk, Kibana/Logstash, Centreon i Windows Device Portal nejsou "jen dashboard". Jsou to řídicí a integrační vrstvy s mimořádně vysokou důvěrou. Jakmile do nich útočník vstoupí, často už neřeší jednu aplikaci, ale celou správní rovinu systému.

## Co mají tyto platformy společné

Na první pohled jde o odlišné produkty. Ve skutečnosti je spojuje několik vlastností.

### 1. Vidí dál než běžná aplikace

Monitoring a admin platforma typicky agreguje:

- logy,
- metriky,
- konfigurace,
- přihlašovací údaje k agentům,
- nebo interní topologii.

Útočník tak po kompromitaci jednoho dashboardu často získá přehled o mnohem širším prostoru než z běžného webového serveru.

### 2. Umí spouštět akce

Tyto systémy nebývají jen read-only. Umí:

- instalovat appky nebo pluginy,
- spouštět kontroly,
- načítat skripty,
- posílat joby,
- nebo spravovat zařízení.

Právě to je dělá tak nebezpečnými. Kdo může ovlivnit "observability" vrstvu, často může nepřímo spustit kód nebo změnit stav jinde.

### 3. Běží s vysokou důvěrou

Typický monitoring server nebo admin rozhraní běží s většími právy než běžná aplikace. Je navržený tak, aby:

- sahal do cizích logů,
- mluvil s interními službami,
- nebo spravoval agenty a zařízení.

To znamená, že i "menší" chyba v jeho přístupu nebo autentizaci bývá mnohem cennější než chyba v běžném frontendu.

## Jak tenhle vzorec vypadal v praxi

### Wall: Centreon jako platforma pro spouštění kontrol

Wall je dobrý příklad toho, že monitoring panel není jen přehled služeb. První vstup vznikl přes slabé heslo `admin / password1` do Centreonu. Samotné přihlášení by ještě nebylo kritické, kdyby Centreon neuměl víc než zobrazit grafy. Jenže on umí definovat a spouštět kontroly, tedy ve skutečnosti distribuovat příkazy.

Jakmile se podařilo autentizovat, authenticated RCE exploit využil právě tuhle řídicí rovinu platformy. To je klíčová lekce: slabé heslo v monitoringu je nebezpečnější než stejné heslo v obyčejném informačním webu, protože za ním stojí systém navržený ke spouštění akcí.

### Shibboleth: Zabbix jako legitimní RCE kanál

[Shibboleth](/shibboleth) ukazuje podobný princip na Zabbixu. První heslo nepřišlo z webové zranitelnosti, ale z IPMI hashe, který se po cracknutí znovu použil pro účet `Administrator` v monitoringu. Jakmile byl admin přístup k dispozici, funkce `system.run` nefungovala jako "pomocná diagnostika", ale jako přímý execution kanál na hostu.

To je důležité zdůraznit: monitoring se nestal nebezpečným kvůli vzdálenému exploitu v Zabbixu. Nebezpečný byl tím, že po kompromitaci účtu už sám obsahoval schopnost spouštět příkazy.

### Doctor: Splunk jako root kanál přes appku

Doctor ukazuje jinou variantu. Splunkd běžel na `8089` a stejné heslo `Guitar123`, které uniklo z Apache logu, fungovalo i pro účet `shaun` ve Splunku. Právě tady je dobře vidět, že observability platforma není izolovaný produkt. Byla navázaná na stejnou identitu a zároveň dovolovala vzdáleně doručit škodlivou appku přes SplunkWhisperer2.

Tedy:

- nejprve unikne heslo z logů,
- pak se z něj stane přístup do Splunku,
- a Splunk se sám stane kanálem pro root shell.

Opět nejde o dashboard. Jde o platformu s instalačním a exekučním modelem.

### Haystack: Kibana a Logstash jako celý řetězec

Haystack je důležitý proto, že zde nestačí mluvit jen o "Kibaně". Ve skutečnosti jde o celý observability řetězec:

- lokální Kibana konzole,
- vlastní file-load primitivum,
- a Logstash workflow, které z log souborů vykonává příkazy jako root.

To je přesně situace, kdy se observability vrstva mění v administrativní rovinu, aniž by to tak na první pohled vypadalo. Na hostu šlo přes `127.0.0.1:5601` načíst vlastní skript a následně zneužít Logstash, který bral řádky `Ejecutar comando:` jako instrukce.

Tento příklad je cenný tím, že ukazuje kombinaci UI, API i backend pipeline. Útok neproběhl "na dashboard". Proběhl na celém řetězci, který dashboard obsluhuje.

### Omni: Windows Device Portal jako správa zařízení

Omni zase ukazuje, že admin platforma nemusí být klasický monitoring stack. Windows Device Portal na Windows IoT Core je plnohodnotné správní rozhraní zařízení. Přes SirepRAT bylo možné spouštět příkazy, a rozhodující mezikrok pak vedl ke skriptu `r.bat`, který průběžně resetoval heslo administrátora.

Tady je dobře vidět, proč má smysl mluvit širším pojmem "admin platformy". Device Portal, Centreon a Splunk vypadají odlišně, ale spojuje je to hlavní:

- sedí nad důvěryhodnou správní vrstvou,
- mají přístup k citlivým akcím,
- a po kompromitaci otevírají víc než jen jeden proces nebo jednu stránku.

## Jak tyto platformy auditovat po footholdu

Po získání prvního shellu nebo přístupu do části prostředí se vyplatí cíleně ptát:

- běží na hostu Splunk, Kibana, Centreon, Device Portal nebo jiná admin platforma,
- jaké porty a rozhraní jsou dostupné jen lokálně,
- jaké pluginy, appky nebo checky lze nahrát nebo měnit,
- a jaké identity nebo tajemství tato platforma používá.

Užitečné jsou hlavně otázky:

- umí tahle platforma něco spustit,
- umí něco nainstalovat,
- a vidí do systémů nebo souborů, do kterých běžný uživatel nevidí?

Právě tam se rozhoduje, zda jde o "jen dashboard", nebo o skutečný admin kanál.

## Jak si to nesplést s příbuznými tématy

Tento článek se dotýká i logů, debug rozhraní a localhost služeb, ale je dobré držet rozdíly.

- Oproti článku o debug a admin rozhraních je tady větší důraz na platformy, které agregují data a spouštějí následné akce.
- Oproti článku o logách jako útokové ploše nejde primárně o samotný log input, ale o systém, který nad daty stojí a dává jim další privilegovaný kontext.
- Oproti localhost článku nejde hlavně o to, že služba běží interně, ale o to, že její role v prostředí je sama o sobě vysoce privilegovaná.

## Obrana a bezpečnější návrh

### 1. Monitoring a admin platformy izolovat jako vysoce citlivou vrstvu

Tyto systémy nemají být chráněné stejně jako běžný interní web. Jejich dopad je větší, protože kombinují viditelnost i možnost zásahu.

### 2. Oddělit identity a nepoužívat reuse hesel

Doctor ukazuje, jak rychle se problém násobí, když stejné heslo funguje v Apache logu, Linux účtu i ve Splunku. U těchto platforem má reuse obvykle vyšší dopad než jinde.

### 3. Tvrdě řídit pluginy, appky a checky

Centreon i Splunk jsou cenné právě proto, že umí rozšíření a akce. Obrana musí přesně hlídat:

- kdo smí nahrávat appky,
- kdo smí měnit kontroly,
- a kdo smí spouštět vzdálené nebo lokální příkazy.

### 4. Nevěřit interním pipeline jen proto, že jsou "pro monitoring"

Haystack dobře ukazuje, že pokud Logstash nebo jiný backend zpracovává vstup jako instrukci, problém není v tom, že jde o observability systém. Problém je v tom, že privilegovaná pipeline interpretuje nebezpečný obsah.

### 5. Admin rozhraní zařízení brát jako plnohodnotnou správní vrstvu

Device Portal, iLO, iDRAC, Centreon nebo obdobné systémy mají mít stejně přísný přístupový model jako nejcitlivější správa infrastruktury.

## Shrnutí klíčových poznatků

- Monitoring, observability a admin platformy nebývají pasivní dashboardy. Jsou to systémy s vysokou důvěrou, které často vidí dál a umí víc než běžná aplikace.
- Wall ukazuje platformu pro spouštění kontrol, Doctor instalaci škodlivé appky ve Splunku, Haystack privilegovanou observability pipeline a Omni správu zařízení přes Device Portal.
- Jakmile se tyto systémy kompromitují, dopad obvykle přesahuje samotný server a sahá do širší správy prostředí.

## Co si odnést do praxe

- Při bezpečnostním review se na monitoring a admin platformy dívejte jako na řídicí vrstvu infrastruktury, ne jako na vedlejší servis.
- Slabé heslo, reuse přístupů nebo podceněný plugin model jsou zde nebezpečnější než v běžné aplikaci, protože za nimi stojí mnohem silnější oprávnění.
- Pokud platforma něco monitoruje, často to zároveň umí i spravovat. Právě tahle kombinace z ní dělá mimořádně hodnotný cíl.
