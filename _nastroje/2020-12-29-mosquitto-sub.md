---
layout: post
author: Tomáš Matějíček
title: "mosquitto_sub"
date: 2020-12-29
tags: nastroje mqtt message-broker localhost topics
---
## Úvod a kontext

`mosquitto_sub` je jednoduchý MQTT subscriber. To samo o sobě zní jako velmi úzký nástroj, ale v praxi má velkou hodnotu na hostech, kde po footholdu narazíte na lokální broker a potřebujete rychle zjistit, co po něm skutečně teče. Není to skener a neřeší exploit. Je to čtečka provozního kanálu, který bývá v interních aplikacích často podceňovaný.

Na [PlayerTwo](/playertwo) právě přes `mosquitto_sub` vyplaval signing key z interního topicu, a tím i cesta k SSH účtu `observer`. To je velmi dobrý příklad toho, proč má tenhle nástroj smysl: broker bývá považovaný za interní plumbing, ale po footholdu se z něj může stát přímý zdroj tajemství.

## Co tenhle nástroj v praxi řeší

`mosquitto_sub` pomáhá hlavně tehdy, když potřebujete:

- připojit se k MQTT brokeru,
- sledovat konkrétní topic nebo celé větve,
- rychle ověřit, jestli se přes broker šíří tajemství, eventy nebo stav systému,
- a získat praktický obraz o tom, jak broker zapadá do aplikace.

To je důležité, protože message broker není jen další otevřený port. Často sedí mezi webem, workerem, podpisovou logikou, firmware pipeline nebo jinou citlivou částí systému.

## Nejčastější scénáře využití v tomto projektu

### Sledování interních topiců po footholdu

Na [PlayerTwo](/playertwo) byl broker dostupný jen lokálně, takže se stal zajímavým až po prvním footholdu jako `www-data`. V tu chvíli stačilo připojit subscriber:

```bash
mosquitto_sub -h localhost -t '$SYS/#'
```

Na tématu `$SYS/internal/firmware/signing` se periodicky objevoval privátní klíč. Praktický dopad nebyl "vidím hezký event". Dopad byl plnohodnotný SSH přístup.

### Rychlé ověření, zda broker nese data nebo řídicí instrukce

U message brokeru je důležité neptat se jen "běží tam MQTT?". Důležitější je ptát se:

- co se v topicu objevuje,
- kdo to publikuje,
- jestli jde o stav, tajemství nebo řídicí instrukci,
- a zda topic žije průběžně, nebo jen při konkrétní akci.

Právě `mosquitto_sub` je na takové první ověření ideální, protože je lehký a dává okamžitou zpětnou vazbu.

## Kdy je `mosquitto_sub` nejlepší volba

`mosquitto_sub` dává největší smysl tehdy, když:

- už máte foothold na hostu nebo jinou cestu k internímu brokeru,
- potřebujete rychle přečíst provoz na MQTT,
- nebo chcete potvrdit, zda broker nese citlivá data.

Naopak sám o sobě neřeší publikování, exploit ani širší mapování celé message architektury. Je to rychlý čtecí nástroj, ne kompletní MQTT framework.

## Nejčastější chyby v praxi

### Představa, že interní broker je jen diagnostika

PlayerTwo ukazuje pravý opak. Broker nesl signing key, tedy provozní tajemství s přímým bezpečnostním dopadem.

### Ignorování localhost boundary po footholdu

To, že broker běží jen na `127.0.0.1`, neznamená nízký dopad. Znamená to jen to, že se stává zajímavým až po prvním přístupu. Právě tenhle pattern rozebírám i v článku [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary).

### Sledování příliš úzkého topicu od začátku

Když ještě nevíte, jak broker vypadá, dává často větší smysl začít široce a teprve pak zúžit na konkrétní větev nebo kanál.

## Související články v projektu

- Rozbory: [PlayerTwo](/playertwo)
- Techniky: [Message brokery a interní fronty jako zdroj tajemství](/techniky/message-brokery-a-interni-fronty-jako-zdroj-tajemstvi), [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary)

## Co si odnést do praxe

- `mosquitto_sub` je praktický nástroj na rychlé čtení MQTT provozu po footholdu.
- Největší hodnotu má na interních brokerech, které přenášejí stav, tajemství nebo instrukce mezi aplikacemi.
- Nejde o exploit. Jde o jednoduchý způsob, jak z lokálního message busu udělat čitelný zdroj informací.
