https://tryhackme.com/room/corridor

   You havee found yourself in a strange corridor. Can you find your way back to where you come?
   In this challenge, you will eplore potential IDOR vulnerabilities. Examiine the URL endpoints you access as you navigate the website and note the gexadecimal values you find (they look an awful lot like a hash, don't they?). This could help you uncover website locations you were not expected to access.

What is the flag?

---


## Reconnaissance 

I opened the application and examined the available doors.

![](images/Pasted%20image%2020261020201231.png)


I noticed that each door had a link, and each URL contained a hexadeximal value that looked like an MD5 hash.

![[Pasted image 20261021190351.png]]

I used Crackstation to identify the values behind the hashes. For example:

![[Pasted image 20261021190514.png]]

I found that the URLs used MD5 hashes of simple numbers. in this case, from 1 to 13. This means that the URL follows this pattern:

```URL
http://10.130.184.193/13
```

The nuber (13) is replaced by its MD5 hash in actual URL.

I then decided to look for other pages that might be accessible.

---

## Enumeration

First, I created a wordlist containing the MD5 hashes of numbers from 0 to 100 so I could use it as a payload in Burp Suite Intruder.

```bash
for i in $(seq 0 100); do echo -n "$i" | md5sum | awk '{print $1}'; done > list_of_hashes.txt  
```

Next, I opened Burp Suite, intercepted a request using Proxy, and sent it to Intruder

I selected the position to use for payload injection.

![[Pasted image 20261021191248.png]]

I selected the Simple list payload type and added the wordlist I had created earlier.

Alternatively, I could simplify the process by configuring Intruder as shown below.

![[Pasted image 20261021191446.png]]

I clicked "Start attack"and examined the results.

One response had a different length from the others, so I opened the page and checked its contents.

I found the flag:

![[Pasted image 20261021191655.png]]

---


## **Summary**

1. **Reconnaissance** — Identified MD5 hashes in the URLs and determined that they represented numbers from 1 to 13.
2. **Enumeration** — Generated a wordlist of MD5 hashes for numbers from 0 to 100 and used Burp Suite Intruder to discover additional pages.
3. **Exploitation** — Accessed a page identified through response-length differences and retrieved the flag.
