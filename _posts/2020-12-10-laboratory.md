---
layout: post
author: Tomáš Matějíček
title: "Laboratory"
date: 2020-12-10
tags: linux ssh exploit enumeration privesc hackthebox
---
## Úvod a kontext

Laboratory spojuje dvě odlišné chyby do jednoho čistého řetězce. Veřejně dostupný GitLab na `git.laboratory.htb` je zranitelný při zpracování obrázků a dovolí číst citlivé soubory z hostu. Lokální root pak už nesouvisí s GitLabem, ale s privilegovaným wrapperem `docker-security`, který slepě důvěřuje proměnné `PATH`.

Právě v tom je ten stroj poučný. První část není „web shell přes GitLab“, ale únik interních tajemství a klíčů. A druhá část připomíná, že i zdánlivě neškodný pomocný skript nad Dockerem se při špatném volání binárek změní v přímý root vektor.

## Počáteční průzkum

### Web a vedlejší git subdoména

Už první `nmap` ukáže, že web na portu `443` používá certifikát se SAN `git.laboratory.htb`. To je důležitá indicie, protože vedle hlavní stránky tak rovnou vychází i vývojová služba, která bude pro celý řetězec rozhodující.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```text
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1
80/tcp  open  http     Apache httpd 2.4.41
443/tcp open  ssl/http Apache httpd 2.4.41
| Subject Alternative Name: DNS:git.laboratory.htb
```

## Analýza zjištění

### GitLab a zpracování obrázků

Jakmile je známý `git.laboratory.htb`, další rozumný krok vede do GitLabu. Na Laboratory běží zranitelná verze, která při zpracování obrázků přes ExifTool/DjVu dovolí vyčíst citlivé soubory z hostu. Tohle je důležitý detail: nejde o shell v kontextu GitLabu, ale o arbitrary file read nad systémem a GitLab konfigurací.

Právě touto cestou se podaří vytáhnout interní tajemství, mimo jiné i obsah souborů jako `/opt/gitlab/embedded/service/gitlab-rails/config/secrets.yml`. Z interních dat GitLabu pak vyjde deploy key, který je znovu použitý jako SSH klíč účtu `dexter`.

### Reuse deploy key jako systémový SSH klíč

To je klíčový mezikrok celé user části. Samotný GitLab foothold by bez dalšího byl omezený na vývojové prostředí. Reuse soukromého klíče ale převádí aplikační únik na plnohodnotný systémový přístup přes OpenSSH. Nejde tedy o chybu SSH, ale o špatné oddělení tajemství mezi aplikací a hostem.

## Získání přístupu

### SSH jako `dexter`

Jakmile je deploy key k dispozici, nejčistší další krok je přepnout se na stabilní SSH session. Tím odpadají omezení webové aplikace a otevírá se prostor pro lokální enumeraci hostu.
```bash
ssh -i id_rsa dexter@10.10.10.216
cat user.txt
```

## Eskalace oprávnění

### `docker-security` a hijacking přes `PATH`

Lokální root už nesouvisí s GitLabem. Účet `dexter` může se zvýšenými právy spouštět skript `docker-security`. Problém není v Dockeru samotném, ale ve wrapperu: interně volá `docker` bez absolutní cesty, takže při běhu spoléhá na `PATH` převzatý z neprivilegovaného prostředí.

To je klasická chyba v delegaci oprávnění. Stačí připravit vlastní program nebo shell script jménem `docker`, dát ho do adresáře na začátek `PATH` a pak spustit wrapper znovu. `sudo` pak jako root zavolá útočníkův podvržený binární soubor.
```bash
cat > /tmp/docker <<'EOF'
#!/bin/sh
/bin/sh
EOF
chmod +x /tmp/docker
PATH=/tmp:$PATH sudo /usr/local/bin/docker-security
cat /root/root.txt
```

## Shrnutí klíčových poznatků

- Laboratory je dvoukrokový řetězec: GitLab nejdřív vydá citlivá data a teprve z nich vznikne systémový SSH přístup.
- Rozhodující nebyla „RCE v GitLabu“, ale únik deploy keye z interních dat a jeho reuse pro účet `dexter`.
- Root část ukazuje starý, ale stále velmi reálný problém: privilegovaný wrapper, který nevolá binárky absolutní cestou, je snadno zneužitelný přes `PATH`.

## Co si odnést do praxe

- GitLab a podobné vývojové služby je potřeba chránit stejně přísně jako produkční aplikaci. Jakmile vydají interní tajemství nebo klíče, útočník často přeskočí rovnou na host.
- SSH klíče nesmějí být sdílené mezi aplikací, CI/CD a lokálními účty. Laboratory dobře ukazuje, že jeden uniklý deploy key může být ve skutečnosti plnohodnotný systémový klíč.
- Privilegované wrappery musí používat absolutní cesty k binárkám a minimální prostředí. Pokud root skript důvěřuje `PATH`, je to ve výsledku skoro totéž jako spouštět útočníkův shell přímo.
