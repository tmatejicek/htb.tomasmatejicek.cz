---
layout: post
author: Tomáš Matějíček
title: "Konfigurace síťových zařízení jako zdroj reuse hesel"
date: 2020-12-03
tags: network credentials password-reuse configuration winrm
---
## Úvod a kontext

Konfigurace routerů, switchů a firewallů se často berou jako čistě síťařský artefakt. Typická představa je, že únik configu ohrožuje hlavně samotné zařízení: někdo uvidí ACL, routování nebo názvy VLAN. V praxi ale bývá hodnota configu mnohem širší. Jakmile export obsahuje:

- lokální účty,
- `enable secret`,
- reverzibilně uložená hesla,
- VPN tajemství,
- fallback účty pro helpdesk,
- nebo provozní komentáře a názvy správců,

nejde už jen o síťovou konfiguraci. Jde o credential dump jiné vrstvy infrastruktury.

Heist je velmi čistý příklad právě proto, že žádný router není konečný cíl útoku. Cisco konfigurace je jen zdrojem hesel, která se později použijí proti doménovým účtům a WinRM. To je přesně hranice, kterou je potřeba chápat správně: únik síťové konfigurace je často identitní problém, ne jen problém síťového týmu.

## Proč jsou config exporty bezpečnostně citlivé

Síťové konfigurace mají několik vlastností, které z nich dělají výjimečně cenný zdroj tajemství:

- bývají ukládané hromadně v helpdesku, ticketu, wiki nebo záloze,
- dlouho přežívají i po změně zařízení,
- spravuje je více týmů než jen čistý networking,
- a často obsahují hesla ve formátu, který není skutečně bezpečný.

To je důležitý rozdíl proti běžnému password dumpu. Uživatel obvykle nečeká, že jeho doménové heslo skončí v exportu routeru. Jenže v praxi se do konfigurací dostávají:

- lokální administrátorské účty,
- hesla pro monitoring,
- tajemství pro integraci s dalšími systémy,
- a hlavně reuse mezi síťovou vrstvou a běžnou správou.

Právě poslední bod bývá rozhodující. Jakmile správce použije stejné heslo pro zařízení, support portál a Windows účet, konfigurace se mění v první foothold pro úplně jinou službu.

## Heist: od Cisco configu k WinRM

Na Heist byl prvním relevantním artefaktem support portál se zveřejněnou Cisco konfigurací. Ta obsahovala několik různých forem tajemství:

```text
enable secret 5 $1$pdQG$o8nrSzsGXeaduXrjlvKc91
username rout3r password 7 0242114B0E143F015F5D1E161713
username admin privilege 15 password 7 02375012182C1A1D751618034F36415408
```

Po zpracování z ní nevypadla jen jedna hodnota, ale rovnou několik použitelných hesel:

```text
stealth1agent
$uperP@ssword
Q4)sJu\Y8qz*A3?d
```

To je klíčový moment. Samotný config ještě nedává shell. Dává ale sadu tajemství, která lze přenést do další vrstvy prostředí.

Na Heist pak dávalo smysl:

1. použít první platný credential k doménové enumeraci,
2. vytáhnout seznam uživatelů přes `lookupsid.py`,
3. a nad nimi pustit opatrný password spray.

Právě tím vznikla platná kombinace pro účet `Chase`, která už otevřela WinRM.

## Různé formy tajemství v síťové konfiguraci

Je užitečné neslévat všechny řádky s hesly dohromady. Praktický dopad závisí na tom, jaký typ tajemství unikl.

### `enable secret`

`enable secret` řeší privilegovaný režim na zařízení. Samo o sobě nemusí být reuse heslem pro další systémy, ale v prostředí se slabou credential hygienou to tak bývá. Navíc jde často o heslo, které zná více adminů a které se dlouho nemění.

### Lokální účty zařízení

Řádky typu:

```text
username admin privilege 15 password ...
```

jsou ještě citlivější. Už nejde o společné „enable“ heslo, ale o konkrétní administrátorský účet, který správce snadno znovu použije jinde:

- v helpdesk portálu,
- v RDP,
- ve WinRM,
- v monitoring stacku,
- nebo v jiném infrastrukturním rozhraní.

### Reverzibilně uložená hesla

Na Heist hrála zásadní roli hesla typu 7. To je důležitá obranná lekce: ne všechno, co vypadá jako „zakódované heslo“, je skutečný hash. Některé historické formáty jsou spíš obfuscation než ochrana.

Z pohledu útoku to znamená, že konfigurace někdy neobsahuje materiál k dlouhému cracking procesu. Obsahuje téměř přímo heslo v čitelné podobě, jen s mezikrokem dekódování.

## Proč reuse mezi zařízeními a doménou vzniká

Z provozního hlediska to není těžké pochopit. Síťové prostředí často používá:

- break-glass účty,
- lokální fallback přístupy,
- ručně psané konfigurace,
- exporty do tiketů při řešení incidentu,
- a sdílené správcovské tajemství napříč týmy.

V takovém prostředí hesla přirozeně migrují mezi vrstvami:

- router -> support portál,
- firewall -> VPN účet,
- switch -> lokální Windows admin,
- monitoring -> databáze -> shell.

Heist je přesný příklad toho, jak se síťové heslo nepoužije k převzetí routeru, ale k přihlášení do úplně jiné identity vrstvy.

## Jak podobné řetězce hledat systematicky

Když narazíš na konfiguraci síťového zařízení, smysl dává nejít jen po jediné otázce `jaké heslo z ní dostanu`, ale spíš po tomto sledu:

### 1. Určit, jaký druh tajemství config obsahuje

- lokální účet,
- sdílené privilegované heslo,
- reverzibilně uložené heslo,
- API tajemství,
- SNMP community,
- VPN pre-shared key.

### 2. Odhadnout, kdo všechno s ním pracuje

Čím širší provozní okruh, tím vyšší šance na reuse. Heslo, které zná helpdesk, network tým i systémový administrátor, málokdy zůstane omezené jen na jedno zařízení.

### 3. Hledat návaznou identitní vrstvu

Prakticky to znamená:

- seznam doménových účtů,
- admin portál,
- WinRM nebo SSH,
- monitoring,
- VPN,
- nebo interní helpdesk.

### 4. Testovat reuse opatrně, ne hlučně

Cílem není slepě zkoušet heslo proti všemu. Cílem je najít místo, kde dává reuse technicky i provozně největší smysl.

Na Heist to nebyl router. Byl to Windows účet v doméně.

## Kde se konfigurace typicky ztrácí

Síťové konfigurace nebývají vystavené jen přes samotné zařízení. Často unikají mnohem prozaičtěji:

- jako příloha v ticketingu,
- v interním wiki exportu,
- v Git repozitáři,
- v záloze na share,
- přes monitoring nebo NMS platformu,
- nebo jako historický soubor zapomenutý ve webrootu.

To je další důvod, proč je potřeba s nimi zacházet jako s plnohodnotným tajemstvím. Útočník se k nim nemusí dostat přes Cisco exploit. Často mu stačí slabě chráněný podpůrný systém.

## Obrana a hardening

### 1. Považovat config export za citlivý secret material

Konfigurace zařízení nesmí být braná jako „jen technická dokumentace“. Pokud obsahuje účty, hesla nebo přístupové klíče, patří pod stejný režim ochrany jako jiné credential store.

### 2. Omezit reuse mezi síťovou a identitní vrstvou

Síťové účty, lokální administrace zařízení a doménové účty nemají sdílet stejná hesla. Jakmile tohle pravidlo padne, export routeru se může změnit ve WinRM nebo VPN foothold.

### 3. Odstranit historické a reverzibilní formáty ukládání

Pokud platforma stále používá formáty, které jsou snadno dekódovatelné nebo crackovatelné, je potřeba je považovat za prakticky kompromitované při každém úniku konfigurace.

### 4. Kontrolovat, kam configy putují

Ticketing, monitoring, wiki a repozitáře bývají slabší článek než samotné zařízení. Je potřeba auditovat:

- kdo exporty ukládá,
- kdo je může číst,
- jak dlouho tam zůstávají,
- a zda se automaticky nereplikují do dalších systémů.

### 5. Detekovat stahování a hromadné exporty

U support a NMS platforem dává smysl sledovat:

- bulk export konfigurací,
- neobvyklé čtení historických ticketů,
- a přístup k archivním configům mimo běžnou provozní roli.

## Shrnutí klíčových poznatků

- Konfigurace síťových zařízení není jen síťařský artefakt. Často obsahuje tajemství, která mají hodnotu i mimo samotné zařízení.
- Heist ukazuje celý řetězec: support portál -> Cisco config -> dekódovaná hesla -> doménová enumerace -> WinRM.
- Největší riziko nevzniká jen z existence hesla v configu, ale z jeho reuse mezi síťovou, aplikační a identitní vrstvou.
- Historické a reverzibilní formáty ukládání dělají z úniku konfigurace prakticky okamžitý credential leak.

## Co si odnést do praxe

- Když unikne router nebo switch config, neptej se jen `ohrožuje to zařízení`. Ptej se i `kde všude může stejné heslo ještě žít`.
- Export konfigurace do ticketu nebo wiki je bezpečnostní rozhodnutí, ne administrativní detail.
- Reuse hesel mezi síťovou infrastrukturou a doménou dělá z podpůrného artefaktu přímý vstup do identity vrstvy.
