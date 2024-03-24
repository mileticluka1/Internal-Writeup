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

While looking at the index page of blog
