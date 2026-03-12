---
layout: post
author: Tomáš Matějíček
title: "PlayerTwo"
date: 2020-12-29
tags: linux rce ssh php exploit enumeration
---

## Úvod a kontext

PlayerTwo je stroj z Hack The Box, který staví na netypické kombinaci Twirp RPC, protobuf definic a následného pivotu přes aplikační tajemství uložená na serveru. Prakticky je zajímavý hlavně tím, že část attack surface si aplikace sama zdokumentuje veřejně dostupným `.proto` souborem.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT     STATE SERVICE VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 0e:7b:11:2c:5e:61:04:6b:e8:1c:bb:47:b8:4d:fe:5a (RSA)
|   256 18:a0:87:56:64:06:17:56:4d:6a:8c:79:4b:61:56:90 (ECDSA)
|_  256 b6:4b:fc:e9:62:08:5a:60:e0:43:69:af:29:b3:27:14 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
8545/tcp open  http    twirp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

=> twirp_invalid_route
=> https://github.com/twitchtv/twirp
=> https://developers.google.com/protocol-buffers/docs/proto3

=> http://player2.htb/proto/generated.proto
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
curl --location "http://product.player2.htb/api/totp.php" \
```
```
	 --header "Content-Type:application/json" \
     --cookie "PHPSESSID=van1nvsd6ugc4jfkgn12qb9ehn" \
     --data '{"action": 0}' \
     --verbose

=> http://product.player2.htb/protobs/

? Jak dostat exploit do firmware ?

wfuzz --hh 26 -w /usr/share/wordlists/dirb/big.txt -X "POST" -b 'PHPSESSID=dv0got03i77dqdbf4jpk8qd3tb' -d '{action=FUZZ}' http://product.player2.htb/api/totp

./dirsearch/dirsearch.py -u http://player2.htb/vendor/ -e php -x 403 -w /usr/share/wordlists/wfuzz/general/megabeast.txt

view-source:http://product.player2.htb/totp

./dirsearch/dirsearch.py -u http://product.player2.htb/ -e php -x 403

./dirsearch/dirsearch.py -u http://player2.htb/proto -e proto -x 403
```

### Enumerace webu (2)

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
dirb http://product.player2.htb/
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Klíčovým artefaktem byl soubor `generated.proto`, který v podstatě dokumentoval Twirp endpoint `/twirp/twirp.player2.auth.Auth/GenCreds` na portu `8545`. Právě tímto RPC voláním šlo získat platné přihlašovací údaje pro `product.player2.htb`, takže následná enumerace produktové části už probíhala z autorizovaného kontextu, ne naslepo.

Další důležitý krok vedl přes `api/totp.php`, odkud šlo získat záložní kód pro druhý faktor a dokončit přihlášení do produktové části. Ta zpřístupnila dokumentaci, firmware ke stažení a stránku pro kontrolu podpisu, takže další směr už byl zřejmý: pokusit se upravit firmware a vrátit přes něj kód na server.

Kontrola podpisu byla chybná, takže šlo nahrát upravený firmware s PHP reverse shellem a získat první foothold jako `www-data`. Ten ale ještě nestačil na stabilní přístup. V systému běžel Mosquitto a interní MQTT komunikace obsahovala SSH soukromý klíč uživatele `observer`, přes který už šlo přejít na stabilní SSH shell a ověřit `user.txt`.

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

Root část už nepatřila webu, ale lokálním oprávněním a doprovodné službě. Zneužitelný SUID helper se zde řetězil s tajemstvím získaným z interní konfigurace nebo zpráv na systému. Samotný SUID program tedy nebyl samospasitelný; rozhodující bylo, že důvěřoval datům, která mohl neprivilegovaný uživatel po předchozím footholdu ovlivnit.

Uživatelský účet `observer` měl k dispozici SUID binárku `Protobs`, která běžela jako root. Reverzní analýza ukázala heap/off-by-one chybu při práci s konfigurací, takže šlo exploit připravit offline a následně jej spustit z SSH relace. Root část tedy stála na kombinaci stabilního přístupu jako `observer` a zneužitelné SUID binárky, ne na dalším webovém payloadu.

## Shrnutí klíčových poznatků

- `generated.proto` zdokumentoval útok lépe než běžný webový recon a dovedl přímo k platným credentialům.
- User část stála na řetězci `GenCreds` -> TOTP backup code -> chybná validace firmwaru -> MQTT únik SSH klíče.
- Root fáze už byla čistě lokální a opírala se o SUID binárku `Protobs` s paměťovou chybou.

## Co si odnést do praxe

- RPC a protobuf definice zpřístupněné bez omezení výrazně usnadňují reverzní analýzu autentizační a obchodní logiky.
- Firmware update workflow musí mít skutečně důvěryhodnou kontrolu podpisu, jinak se z něj stává přímý RCE kanál.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Interní message bus nebo MQTT témata nesmí přenášet dlouhodobá tajemství v čitelné podobě.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
