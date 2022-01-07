---
layout: post
title: OpenSSL CLI Tool
date: 2022-01-05 14:00
image: '/assets/img/posts/openssl-cli-tool/sslbanner.png'
tags: [OpenSSL, PKI, PKC, Cryptography, x509, Encryption]
featured: false
---

# 1-INTRODUCTION

**_OpenSSL_** is a command-line utility that facilitates cryptographic operations such as symmetric or asymmetric encryption, public-key
cryptography, hash functions, digital signatures etc. You can download the source code from [http://www.openssl.org](http://www.openssl.org/).
This guide will cover just the basic functionalities of the OpenSSL.
<br><br>After the installation check for the version.
{% highlight bash %}
root@bbsec:~$ openssl version
OpenSSL 1.1.1m  14 Dec 2021
{% endhighlight %}<br>
OpenSSL contains various commands. Here is the list of them.
{% highlight bash %}
root@bbsec:~$ openssl help
Standard commands
asn1parse         ca                ciphers           cms               
crl               crl2pkcs7         dgst              dhparam           
dsa               dsaparam          ec                ecparam           
enc               engine            errstr            gendsa            
genpkey           genrsa            help              list              
nseq              ocsp              passwd            pkcs12            
pkcs7             pkcs8             pkey              pkeyparam         
pkeyutl           prime             rand              rehash            
req               rsa               rsautl            s_client          
s_server          s_time            sess_id           smime             
speed             spkac             srp               storeutl          
ts                verify            version           x509 
{% endhighlight %}<br>

Brief descriptions for some parameters:
- **ca**: Create certificate authorities.
- **dgst**: Compute hash functions.
- **enc**: Encryption/decryption via secret key algorithms such as a secret key stored in a file or using a password.
- **genrsa**: Create a pair of private/public keys by using the RSA algorithm.
- **password**: "Hashed password" generator.
- **pkcs7**: Management tools for PKCS#7 standard.
- **pkcs12**: Management tools for PKCS#12 standard.
- **rsa**: Data management for RSA.
- **rsautl**: Sign/Verif a digital signature or encrypt/decrypt with RSA.
- **rand**: Pseudo-random bit string generator.
- **verify**: X.509 digital certificate verifier.
- **x509**: Data management for X509.


# 2-SECRET KEY ALGORITHMS

It is possible to implement various types of secret key algorithms with OpenSSL. Here is the list of them.

{% highlight bash %}
root@bbsec:~$ openssl help
...
...
...
Cipher commands:
aes-128-cbc       aes-128-ecb       aes-192-cbc       aes-192-ecb       
aes-256-cbc       aes-256-ecb       aria-128-cbc      aria-128-cfb      
aria-128-cfb1     aria-128-cfb8     aria-128-ctr      aria-128-ecb      
aria-128-ofb      aria-192-cbc      aria-192-cfb      aria-192-cfb1     
aria-192-cfb8     aria-192-ctr      aria-192-ecb      aria-192-ofb      
aria-256-cbc      aria-256-cfb      aria-256-cfb1     aria-256-cfb8     
aria-256-ctr      aria-256-ecb      aria-256-ofb      base64            
bf                bf-cbc            bf-cfb            bf-ecb            
bf-ofb            camellia-128-cbc  camellia-128-ecb  camellia-192-cbc  
camellia-192-ecb  camellia-256-cbc  camellia-256-ecb  cast              
cast-cbc          cast5-cbc         cast5-cfb         cast5-ecb         
cast5-ofb         des               des-cbc           des-cfb           
des-ecb           des-ede           des-ede-cbc       des-ede-cfb       
des-ede-ofb       des-ede3          des-ede3-cbc      des-ede3-cfb      
des-ede3-ofb      des-ofb           des3              desx              
idea              idea-cbc          idea-cfb          idea-ecb          
idea-ofb          rc2               rc2-40-cbc        rc2-64-cbc        
rc2-cbc           rc2-cfb           rc2-ecb           rc2-ofb           
rc4               rc4-40            seed              seed-cbc          
seed-cfb          seed-ecb          seed-ofb          sm4-cbc           
sm4-cfb           sm4-ctr           sm4-ecb           sm4-ofb           

{% endhighlight %}<br>

The list includes the _base64_ encoding standard. It is not a secret key algorithm since no secret key generated for
base64. This encoding standard converts binary input into alphanumeric characters and vice versa.
{% highlight bash %}
root@bbsec:~$ echo "www.bbsec.net" > plaintext.txt
root@bbsec:~$ openssl enc -base64 -in textfile.txt
YmJzZWNuZXQK
root@bbsec:~$ echo "YmJzZWNuZXQK" > encoded.txt
root@bbsec:~$ openssl enc -base64 -d -in encoded.txt
bbsecnet
{% endhighlight %}<br>
On the contrary, [Advanced Encryption Standard (AES)](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) is indeed a secret key algorithm.
AES algorithm requires a secret password as a key to encrypt data. So let us see how it works.

{% highlight bash %}
root@bbsec:~$ openssl enc -aes-256-cbc -pbkdf2 -in plaintext.txt -out encrtpyed.bin
enter aes-256-cbc encryption password:
Verifying - enter aes-256-cbc encryption password:

root@bbsec:~$ cat encrtyped.bin
root@bbsec:~$ "Salted__W���U�j���í��R'���#��b�"
root@bbsec:~$ hexdump encrtpyed.bin 
0000000 6153 746c 6465 5f5f d357 dfd9 dd55 856a
0000010 aab8 adc3 a881 2752 f6b5 23f5 84a8 fe62
0000020
root@bbsec:~$ file encrtpyed.bin 
encrtpyed.bin: openssl encrypted data with salted password

root@bbsec:~$ openssl enc -d -aes-256-cbc -pbkdf2 -in encrtpyed.bin -pass pass:111111
www.bbsec.net
{% endhighlight %}<br>

# 3-PUBLIC KEY CRYPTOGRAPHY

This section will demonstrate how the OpenSSL manages public key algorithms. Therefore we will use one of the most common
public-key cryptography algorithm RSA.

## 3.1-Generating the Key Pair

First, we need to create an RSA public/private key pair. We will create a 2048 bit key.
{% highlight bash %}
root@bbsec:~$ openssl genrsa -out privkey.pem 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
..............+++++
......................................................................................................+++++
e is 65537 (0x010001)
root@bbsec:~$ cat privkey.pem 
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA4tX0r+2pX9Mssky5Kz9ZNEvDswLjzNgO8cojNA6eHA8VSSGr
XUsbewpH89wAgZzmEk2whV28ktMh7y75HSm3b7D7bpun8ODACLaA5TpsMNdJtrG+
6kjw0Sex6RMOHoLya5Bv2bD+AtqqLNKTo5rBwKPpYs2fPOeTVFomSnyO6S1GiBNM
KD8AdBk7bLS9ejsqyv6j9q2fiL6SDeNFXup9mF3e+lb0wXfpkP7db4m06Ci1zkSb
cv4bDNXS/P0As+1Rjbu9QCFhypUFdfwR1BHl7fYxIY/HCxMXWq0xXlxmGybEGCn7
UJWWgDvqx4PbmueXWIOuAtHJ/wu6HXH43/4ztQIDAQABAoIBAA94tGXDegf1KVlH
7mFKwtTUThbJnav9GJfZR6lnTdVwGe2RBFUqqEcuHlY9rTMp9m9NKTsPd6s0B15+
/7LDg6V0ltGmgD/ntHFjsUrxPyvdo0N4wCLOss4xPOs+x3nBSLOZeGeKsOiU7YJ9
ImDIT2rKQ0Lf73qB+QSJ2Y6/DChPM17d4ZnI8gfINlrOLAvl8VAZTX+4HP7wOYAW
m9nwZ8NXvudjk6vFs5m2K2t5iZpANeLMmmTanfM4/vhHBWyVAJfT/ZpBts3Agwdw
RltplK+n/4/G1AzyoQpQcuNmuATSNqiiLuAno+UM6oqY1t31q0jObHpRIfEcS6Bo
8YhTXskCgYEA8jM6p22JYv4WGoyEdiPeVJwMFj2AoTpdqEo19+fkvHhI1MBN8E7d
qJ+mP3wChP2GEmDeW+wOqRAncJlVbh/nfH/t4WWUv6pYIwWfJ8IvgeWzbz8Wdat+
y72ut+Mu5CCIfOmZi9K58+2ObMlNFR87yK3mRsl/fa5X2fJi9i2rb6sCgYEA78Ke
qKSSWM8cjHyB5dvN0dQUtcEZ8EWagJjyX9/tfRond4x6PPQvTtX742I8GdJ6T95s
esatTw+kEIim5wSXsM+wdEtMFfU1n6oEYrMpg+sgyIp72QDQi3kr/j5Dpb4DOAOQ
Hrh5Pbyq/uFFPPzX1aBSphCRKqBLmcyk1X6dCh8CgYEA8Pc9DOSzApVPAnz5MN0A
z2cts1bfSglasxuaVBoX/dcihuEI6eRdLe4gphrIGu5tXI2ZzRSvhU64HpO/ZkBB
vCE/V7gL5SEibT2jmhfd0jvpaO34d3v3O9dtJDDYL0ma4cQ76tvt/B1GTT99/FzF
yyQQ7i59NFqntwQrp0fKv98CgYAfolcauzQQAauroZXmBR1f7RKadJL+j8B17Tg1
jC8ijXvdmyxZtII1bahhdQmnAo1e0mMPw/0D7HViNRWIb6OwEYcfoPu1/feITH9t
omP84t4dd6AlnqTlciRq1D5KtQpprpaqZv6gNa9+F6zyAg5cQl4FSTROIn43Gbg5
7w27UwKBgQDeuuDoYqszBhembtG7sT7+5Anryw5qazYxXND1nw6ikU1T9t50wr4P
oVUmGx78XnzTrLLRXG7YNIRxyih9lk07zyvWpe+4MvLNZt2Fsv68uAFXfRhCex6i
2tuq+ge7jLSujkdU3uN8pV5VPEPs9lWsM9lrmbMjO/GMx0SHd5f49g==
-----END RSA PRIVATE KEY-----
{% endhighlight %}<br>
A single privkey.pem file is created. The private key is encoded with the Privacy Enhanced Email (PEM) standard. 
That file contains both private and public keys. This is not what we want since
the private key must be kept secure while the public key should be distributed publicly since other people needs that public key to 
send encrypted messages to you (or to control if a document or a certificate has been signed by you).
<br><br>The way to extract public key inside from the 
privkey.pem file as follows:
{% highlight bash %}
root@bbsec:~$ openssl rsa -in privkey.pem -pubout > pubkey.pem
writing RSA key
root@bbsec:~$ cat pubkey.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzqi5qVmfLf+lCaWGbZ43
iAb3Hofr9MdIUL6i7lxIkViq8xwF90ThiKdgQ5Qn6heacXmQx0aJ9+F+HqpS47Na
aBz/NyHbzID8tSHwE6LhgS5gSsW2yl4/R5Zj3SMndZdJj9ExX6lesJDesOnzIEt7
6JMnOUinbGm9Lmtu727qBJh0KlNKmK5zUsi5ZxFcfC0onfhGWNbP1floC6KQzdY5
tS3nB7bXeuRTC4qJEYkQoXbQ3U4pLpu4vIdslwfjConBEfA9/3nWyaCXsvsSmAfC
Bl4FQVDSUjWO7mpPUujs2V+wdlH7fWXzkTmtpWOO8XtBQpYIlOqCAChpELnAv6Rf
+wIDAQAB
-----END PUBLIC KEY-----
{% endhighlight %}<br>

The following output indicates the details of the RSA key pair. Those details strictly related to the RSA's algorithm therefore
if you are interested in the math behind the RSA algorithm, you can conduct a further investigation on it.
{% highlight bash %}
root@bbsec:~$ openssl rsa -in privkey.pem -text -noout
RSA Private-Key: (2048 bit, 2 primes)
modulus:
    00:b3:4e:f9:71:fc:3e:af:29:f9:63:a0:38:9b:3f:
    5f:e2:0a:32:32:0c:fa:4f:c7:e7:79:b5:9b:bf:5c:
    87:9a:75:d8:1e:a6:48:09:4f:3f:95:87:aa:d7:a1:
    5d:18:a2:13:27:c4:43:03:6b:8f:e6:81:70:9c:32:
    66:8f:97:32:11:8b:3c:b4:e5:60:5c:02:c9:4a:67:
    76:00:28:68:4c:23:24:63:88:7d:14:8f:9a:6a:56:
    72:03:cd:97:d9:e8:81:9a:af:94:2b:f5:07:ef:4d:
    8b:f9:fd:d5:c7:f2:c7:91:06:89:d3:4f:dd:24:9a:
    72:60:78:aa:7d:ed:e9:07:04:78:64:9a:5f:a2:fa:
    03:24:92:9f:d5:dd:8e:b9:11:f8:dd:99:3f:03:51:
    2a:63:77:2c:d8:89:a6:0c:80:e8:5b:e4:c6:5b:10:
    c0:c8:c4:ca:46:84:c7:df:72:f9:11:7e:c2:29:81:
    e6:4c:86:c0:c4:fe:6c:a0:a7:66:83:45:44:1e:59:
    93:4c:35:fc:fb:00:67:ed:63:a3:66:e3:6b:8b:b7:
    ab:bf:f8:51:1e:af:a4:e1:09:42:41:8f:75:82:cf:
    29:2e:0e:6a:c1:19:62:f1:a0:44:bc:ad:20:06:12:
    d8:a3:fa:d8:a6:a5:5a:c5:bf:f4:9d:1a:b9:b9:5e:
    6c:6f
publicExponent: 65537 (0x10001)
privateExponent:
    00:9d:a6:81:11:23:fb:a5:1c:9d:85:67:78:7d:9e:
    f1:d9:96:a7:5d:74:25:9c:81:a1:56:54:43:74:b3:
    91:12:50:2c:4d:7e:5b:75:bb:f4:a6:ae:da:99:ad:
    e9:61:60:16:c1:6f:00:90:80:40:cc:24:e0:72:a4:
    a9:a1:f4:08:74:7e:5c:48:9c:27:e5:9e:19:86:ce:
    82:64:4f:22:ac:56:75:87:01:99:1f:bb:c6:c3:59:
    ef:f2:c2:0f:91:ea:a8:10:ed:f0:b3:d9:43:39:b6:
    8f:ac:a3:ee:13:57:b4:f9:20:ab:8b:5b:fb:8e:54:
    30:dd:fb:19:c3:90:aa:c2:9c:45:cc:a2:3f:05:57:
    4a:7a:d1:21:99:1e:bc:75:ef:8e:6d:44:48:2c:91:
    5b:2b:c7:56:44:ba:f0:7a:4a:45:de:87:6f:91:c9:
    06:9a:15:eb:3f:7b:dd:aa:47:1e:0e:8f:4c:e7:c5:
    0e:6f:b3:dd:6d:2a:99:8f:f0:8c:e1:48:e6:af:8b:
    29:3d:31:9f:94:4d:5a:76:58:42:d3:49:7d:c9:dd:
    b3:56:d0:09:8b:95:5e:08:ab:94:98:5b:9e:d4:29:
    21:8a:a6:2d:a9:2a:58:da:3f:7f:ce:0e:97:d2:06:
    a1:98:c4:4e:1f:91:05:ad:7a:d0:ab:e1:b2:69:07:
    3d:79
prime1:
    00:d8:9e:75:b1:1e:c2:f2:3f:1e:71:81:d5:6c:fe:
    40:3e:38:0d:30:2e:2c:14:36:3c:d7:09:30:58:04:
    df:dd:e7:54:d2:67:67:3d:c7:79:eb:c4:c4:5b:58:
    1a:0d:db:49:c8:0b:b9:26:19:ae:da:83:29:a3:96:
    d2:c6:df:37:53:33:7e:35:5b:54:4b:32:c2:dc:76:
    f2:8a:99:56:67:87:24:53:7d:3e:aa:90:d0:a7:06:
    70:42:4d:ca:5e:31:52:49:98:c1:08:1b:e5:24:50:
    2f:41:12:d2:f3:1a:c4:c4:c6:08:11:ac:c8:29:08:
    7f:ad:02:f2:99:28:e9:f7:5b
prime2:
    00:d3:e8:11:88:df:ce:96:de:84:0f:ed:e8:52:af:
    cb:e9:6d:0b:67:b5:20:a5:1c:52:17:3c:a6:79:ca:
    65:0c:75:bd:01:d6:ce:cc:74:0d:80:de:3a:81:f5:
    d2:d6:59:84:55:2a:d2:f4:7c:be:74:6e:8d:73:d8:
    47:7c:18:33:22:f4:78:cb:83:43:b0:db:c0:eb:8e:
    cf:d9:27:10:3f:5d:10:c9:79:90:d5:31:33:f8:a2:
    27:3d:77:c5:0a:98:54:2c:89:d1:9e:c4:bf:83:88:
    21:0e:7a:b2:e3:5b:47:a9:f1:9f:21:f7:34:e2:bc:
    16:de:11:94:07:5e:c7:ff:7d
exponent1:
    00:c4:73:38:d3:0f:cd:c6:7a:1d:b6:dd:03:5c:9c:
    5c:50:d0:ee:8c:f2:62:c1:55:ca:e9:4d:89:0d:5a:
    26:58:8d:92:2c:5a:e0:93:73:93:8b:91:60:6e:62:
    c1:06:2e:08:84:a6:b5:1b:eb:90:da:d4:b6:ef:88:
    39:d1:67:e0:39:d1:6a:35:23:85:97:c9:0a:55:7c:
    7e:4b:d9:f2:35:63:a7:3b:1c:4b:b7:ce:2b:9c:3e:
    47:92:aa:0f:cc:4a:b8:90:cc:3a:cb:8a:d8:cd:8c:
    f6:bd:f2:3f:63:7f:b4:51:ac:32:e7:2c:a6:3e:28:
    59:f9:e2:c1:76:cb:57:1c:1f
exponent2:
    2f:04:45:a7:b5:e8:b3:86:c9:8c:73:3f:e1:e0:c9:
    80:90:46:40:8b:6a:a3:d7:c5:cb:0c:14:ef:de:dd:
    4e:c7:6c:d9:54:9c:eb:b6:30:2c:d0:a1:f0:a5:e7:
    52:d1:e7:cf:b1:c1:be:a7:52:e6:a8:84:d0:18:43:
    bc:1f:ee:70:aa:07:87:38:27:b3:bc:fe:70:05:6e:
    ce:82:a1:53:3d:c5:f4:bd:f9:49:a4:32:20:cf:71:
    9f:6c:cc:96:4e:38:16:ed:b9:49:dd:e3:94:3e:86:
    ff:1c:70:46:8b:c1:39:ce:b7:7d:24:c9:62:29:53:
    75:90:36:e4:ef:bd:b6:4d
coefficient:
    00:b6:68:f2:3a:23:18:0d:a1:4e:9d:df:b1:42:8b:
    a4:cb:73:64:45:96:92:df:92:a8:40:7d:5d:f2:bf:
    f6:5f:6d:3e:b9:5f:b2:d1:08:a2:6b:e8:c2:1e:a4:
    76:72:5b:bf:f1:55:41:90:f6:b4:e0:f3:76:84:a1:
    15:91:a4:bf:d9:08:ed:a4:a6:be:42:bf:7c:27:a4:
    1f:81:0f:b6:33:bf:e5:78:a3:a9:43:77:48:ec:ce:
    fc:d4:55:73:a0:58:f8:fb:2b:7c:9c:7b:4e:19:dd:
    50:2d:91:a5:16:cc:68:48:7f:15:b5:2e:14:cb:82:
    41:73:29:70:a0:fc:a5:11:04
{% endhighlight %}<br>
To encrypt the private key and enhance its' security, we can use symmetric key cryptography, such as DES3.
{% highlight bash %}
root@bbsec:~$ openssl rsa -in privkey.pem -des3 -out enc-priv-key.pem
writing RSA key
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
root@bbsec:~$ cat enc-priv-key.pem 
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,00672B83BD258084

tiqDxmmKct026+B6ZcahTuYSgWOjSJvogkjuZc+V8SUo37NOBRWv4Ie7CEfzqOOu
WKzr8gn8WFaRgfI0W+fgsXk8TZ/9yLUB/tVF3kxaM28bItvazJSkk6XLqAsqhJu8
sXhEyi69HlyYeCXdasBjZdoEc+ma5QjGP5VYZ7wZyak6jhxn2ExH+JfkS+3l9uYi
NeD61SLo7unqV44Rl+SDCYxYN6G60Jqv6wuoG8abBadEzo9nXIBMJ2gy0H1uSSqW
ECqRB4V5T4pA9RfuolCl0x76IUWY0ZLRIBLH//pGU+KBG2ZqsFIPg/Fsh/hOfCsL
rGE/rm60wuG7bUs7qbSe0ZjF6Ktu8VdkEfwzezBuhuWSgVy1UBPZxmRzYvO64yuG
k0TLFXzhwxSeR/UJ+GRQAt2e/U6MUw6E2ktUHUAIisyLqJgbU07Xg/nSBn3Lv6aU
5hABRIijp7yAK94HAFOdkliStgO0N3X/ozJyznN/JPW6Cs8GAetiIlyKbGgIxHsH
HOqjS/Zckd82+X3npqU7DFJ6uJ1BwGBQjwTL/r92OVQSp3Rj2sIkj6n087s+iHSm
Gu/w+lBSuIvJyQH8YV/6OmDUHVR6mjOOqP2xZmSN14GxlufW8f7I/iOw4bXRIaV/
PB/VXjjoh8V9VCI8Rcpm9kn/YP75iIfBQR3eYHnVP56C434vDd/moJej7JSJcxbJ
kGPo6l35XDKhZ2rekr0ePiinhYY/BK98MsYIYxxLvL55QrjHI2H0RaNmp71UDQG9
UL1d9sAWyKXSIjF2HKCRUm1+q/r9M8OLSCQBaOZ0H716dvtr7bZ+AXACO4JSYxCT
Un2LMxM78hJ3If5Kb1C/k1OCV/BVJbVaU/ySQ+OzOVncLHYBG7jm7Ezuq5DT6v/O
1KPX4/jomd2C4ZZiBK/G4hV6MT0cVFhJv+ThKLqekXQuNkbcAojf2ndowwxkeCye
nU61PISBXI7uBLmCuJEgQy4THWbCMKgL+nUHLrshwJwyaHkzXG2MMjTcOZj0RTpZ
Il3zImidoFR7JEuBl6hkJlZvljmUSIf1+hZUnYYQWhEoPx3SXVanIm/rJrgaxTxC
k7m15T0DGdhEnAcebYChx11hH/jyRuAz31LcrgzVFfvYFJHKkboAgAc7Zoi1gU+0
mDLnsW/18jWcMO6FU6+Mh5XUWoWe7tMerwYDfZKJuumxii2ufNRZXiPoLoDq4em7
d9uQspcsr+IQG7ByB+l9Y+Lk6ymPlMR34afSnG5qHUSmgB01zyV7ov1beyivOdyl
Ja9DQCuR0KcA1HvLCa1G94QiQ2AHTAsS3mBA8rDGXqOls76lmDF+m2J+7+6XB+e3
DZstVBKti0VCfZnY4l5gW+LGCUup5OUn/NJrAT3veh9TjbxSWEd2ZfA7fdS0zfjP
d7xal0w7v30eD6l4Vx7HXjm6KF9VH+uLL41exWn/8t1LmqGIKzPwRCNVOrN6NWg7
vuMKYMuBl5yza/vfMdeGIlL4fags+EMVC3yz4+xKtHTB/aRRcigNB6hBsrCo8L+S
C1Yr6FdrosL12DFP3UFFvva58jwR+EkGQfSUjZmY67PFLcn93bzkTfV8hmPFjy9j
-----END RSA PRIVATE KEY-----

{% endhighlight %}<br>
Since the private key is encrypted with DES3, you must decrypt the private key back with DES3's passphrase when you 
want to decrypt messages encrypted with your public key.

## 3.2-Encryption

Since we have created the RSA key pair, we can produce a digital signature or perform encryption.
{% highlight bash %}
root@bbsec:~$ echo "this file will be encrtyped with rsa" > plaintext.txt
root@bbsec:~$ cat plaintext.txt
this file will be encrtyped with rsa
root@bbsec:~$
root@bbsec:~$ openssl rsautl -encrypt -in plaintext.txt -pubin -inkey pubkey.pem > enc-text.txt
root@bbsec:~$ cat enc-text.txt 
"@X|aTvr
        3�K���G-U��qz�F��YXm8��4=v��$6w^��̫<�����KH�]��
                                                      ��
��E��w-|��h;}[ݡ;)77��u��?��ޣ���Ù���P�t'�T�" 
{% endhighlight %}<br>
To decrypt the encrypted txt file, use the private key.
{% highlight bash %}
root@bbsec:~$ openssl rsautl -decrypt -in enc-text.txt -inkey privkey.pem 
this file will be encrtyped with rsa
{% endhighlight %}<br>

## 3.3-Digital Signatures

In this section, we will be creating a digital signature and verifying it. Since RSA can only encrypt data smaller than or equal to
the key length, first, we compute the digest of the data to sign it. Note that in practice, the procedure is a bit more complex.
The way of doing it in the real world is to implement a schema called [_RSA-PSS_](https://crypto.stackexchange.com/questions/57607/what-is-rsa-pss-and-how-is-it-different-from-a-hash).

### 3.3.1-Creating the data file then digesting it
In this instance, I will use the SHA1 hash function to digest the data.
{% highlight bash %}
root@bbsec:~$ echo "this document will be signed" > plaintext.txt
root@bbsec:~$ openssl dgst -SHA1 plaintext.txt > plaintext-sha1.txt
root@bbsec:~$ cat plaintext-sha1.txt
SHA1(plaintext.txt)= dc2fe0f7a744a48ded8ce61710c385457e35296a #Data's digest
{% endhighlight %}
### 3.3.2-Signing the digest with the private key
{% highlight bash %}
root@bbsec:~$ openssl rsautl -sign -in plaintext-sha1.txt -out signedfile -inkey privkey.pem 
root@bbsec:~$ cat signedfile 
"�9�O��N�q�k������Kc��ګ�4ci��GC����6�vq���2�K�iy���Q�s���!*��P�����!M��s���/o�,�b���$iJ���������s��?4�!�a"
{% endhighlight %}

### 3.3.3-Verifying the digital signature
{% highlight bash %}
root@bbsec:~$ openssl rsautl -verify -in signedfile -pubin -inkey pubkey.pem 
SHA1(plaintext.txt)= dc2fe0f7a744a48ded8ce61710c385457e35296a #Decrypted signature
{% endhighlight %}
As you can tell, the digest of the data and the result of the decrypted signature validates each other. That means signature is valid.

# 4-CREATING DIGITAL CERTIFICATES (PKI)

If you are not familiar with the _Public Key Infrastructure_, you can read it [here](https://www.bbsec.net/2022/01/01/public-key-infrastructure/).
## 4.1-Generate a key pair

{% highlight bash %}
root@bbsec:~$ openssl genrsa 2048 > myprivatekey.pem
Generating RSA private key, 2048 bit long modulus (2 primes)
..........+++++
..................................................................+++++
e is 65537 (0x010001)
root@bbsec:~$ openssl rsa -in myprivatekey.pem -pubout > mypublickey.pem
writing RSA key
{% endhighlight %}

## 4.2-Generate a CSR
{% highlight bash %}
openssl req -new -key myprivatekey.pem > cert_req.csr

Country Name (2 letter code) [AU]:tr
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:Istanbul
Organization Name (eg, company) [Internet Widgits Pty Ltd]:BBSec
Organizational Unit Name (eg, section) []:Cyber-Sec
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:imbarisburak_buisiness@protonmail.com
A challenge password []:111111
An optional company name []: 
{% endhighlight %}
## 4.3-Act as our own CA to self-sign our own certificate
{% highlight bash %}
openssl x509 -req -in cert_req.csr -signkey myprivatekey.pem > signed.cer
Signature ok
subject=C = tr, ST = Some-State, L = Istanbul, O = BBSec, OU = Cyber-Sec, emailAddress = imbarisburak_buisiness@protonmail.com
Getting Private key
{% endhighlight %}
## 4.4-Display all the information in the certificate
{% highlight bash %}
root@bbsec:~$ openssl x509 -in signed.cer -text -noout
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            34:1c:a6:c0:0d:06:8f:2c:58:46:ae:1b:17:f1:53:b6:27:52:6c:f8
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = tr, ST = Some-State, L = Istanbul, O = BBSec, OU = Cyber-Sec, emailAddress = imbarisburak_buisiness@protonmail.com
        Validity
            Not Before: Jan  5 19:52:19 2022 GMT
            Not After : Feb  4 19:52:19 2022 GMT
        Subject: C = tr, ST = Some-State, L = Istanbul, O = BBSec, OU = Cyber-Sec, emailAddress = imbarisburak_buisiness@protonmail.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:d4:4c:57:45:97:1b:30:d8:fa:b8:5d:52:9f:be:
                    21:d8:d2:11:4b:34:bc:55:53:00:5c:e9:00:9b:87:
                    2d:6b:b1:f8:9e:3a:75:fb:96:9a:31:41:c8:2a:5e:
                    54:10:20:9f:1c:36:4d:b2:6c:66:4e:80:d3:04:19:
                    9f:3a:1a:21:3b:d1:f5:cf:5f:a4:06:6c:b5:38:bb:
                    01:8f:1f:51:82:67:6a:32:01:2d:06:83:03:79:95:
                    42:63:1a:01:87:9e:bc:17:ba:1c:03:1c:a0:92:dd:
                    56:19:a0:eb:38:42:79:43:4b:f2:d8:28:9d:f6:5b:
                    59:38:27:b7:bc:e2:23:31:57:5f:9c:46:36:00:4c:
                    b9:a9:ba:1c:74:73:3c:ad:35:ed:60:02:03:53:28:
                    44:74:c9:e9:f9:05:7f:c1:9e:57:7e:09:c3:41:17:
                    5c:77:36:08:92:64:d4:73:e8:d1:eb:79:57:dd:04:
                    b8:b4:f2:20:f0:42:df:03:68:a3:ca:f0:69:66:a3:
                    46:d4:cd:69:cd:2b:32:8e:5d:06:5c:b8:b0:b5:78:
                    3e:72:1d:cf:95:78:ea:ea:85:60:05:0c:e6:0f:3b:
                    06:3f:78:54:7f:bb:23:d8:1b:25:d8:60:ef:a2:ee:
                    21:4a:4d:22:3b:80:6f:3b:b7:42:8e:86:66:aa:67:
                    54:c9
                Exponent: 65537 (0x10001)
    Signature Algorithm: sha256WithRSAEncryption
         8c:6f:7a:b2:a7:ed:96:ed:7c:26:93:6d:4e:6c:07:c4:1b:76:
         7e:ef:08:1a:4e:15:f8:35:38:dd:88:d3:67:1a:b7:56:ee:7b:
         6e:8a:a9:e3:05:2a:69:55:3c:ec:8b:c1:45:c4:dd:97:54:e6:
         5b:c6:e4:b0:ea:b2:00:e7:e1:f6:d2:46:b2:65:67:e1:d8:bb:
         f9:89:c1:93:05:7c:6a:8c:ef:c9:76:7a:e8:3e:cc:90:21:25:
         f3:c5:4a:9f:87:18:3c:cf:50:2a:7a:1e:b8:49:65:0e:d2:a0:
         d4:cb:8b:27:5a:40:76:ee:52:30:2a:3a:70:63:2d:97:a7:31:
         49:96:60:d9:d9:17:35:02:57:f0:cd:1f:b0:54:15:49:e3:5f:
         1d:19:fd:3e:ff:03:58:f5:cd:ec:f3:a9:9e:e5:ff:b9:bc:d8:
         3d:f3:0a:4f:f5:18:5b:b8:86:4a:cb:6f:89:40:f7:3e:f5:f0:
         6c:be:d0:fd:54:91:43:0f:aa:8a:58:b7:f9:80:c9:2c:07:38:
         f6:6b:18:54:e7:d5:00:8d:ad:36:0f:47:6f:01:d6:2c:79:1b:
         04:65:61:21:1d:e7:b2:72:5b:a5:ab:01:ca:f4:33:7e:26:69:
         e9:4f:dd:f9:c9:60:27:60:a2:0c:98:2d:d4:c9:28:c3:c7:5b:
         f2:51:34:b9
{% endhighlight %}
## 4.5-Put our certificate and key in a PKCS #12 container
{% highlight bash %}
root@bbsec:~$ openssl pkcs12 -export -in signed.cer -inkey myprivatekey.pem -out p12cert.p12
Enter Export Password:
Verifying - Enter Export Password:
{% endhighlight %}
## 4.6-Display information in a PKCS12 file
{% highlight bash %}
root@bbsec:~$ openssl pkcs12 -info -in p12cert.p12 
{% endhighlight %}
## 4.6-Create a key and a self-signed certificate in one command
{% highlight bash %}
openssl req -new -x509 -newkey rsa:2048 -out new_cert.cer -keyout new_key.pem
{% endhighlight %}
