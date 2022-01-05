---
layout: post
title: Public Key Infrastructure
date: 2022-01-01 16:00
description: Explaining all the terminologies and principles behind Public Key Infrastructure (PKI).
image: '/assets/img/posts/public-key-infrastructure/2.jpg'
featured: false
---

This post will discuss all the terminologies behind the **_Public Key Infrastructure (PKI)_**.
It is essential to have knowledge of _Public Key Cryptography_, _Encryption_ and _Digital Signing_ as the prequisition.

# WHAT IS PKI?
A **_Public Key Infrastructure (PKI)_** is a collection of roles, policies, hardware, software, and procedures that are 
required to produce, regulate, disperse, use, store, and revoke x.509 certificates along with manage public-key encryption.
# WHY PKI IS IMPORTANT?
While public-key encryption solves the problem of the data's legibility by everyone, PKI adds authentication mechanism on top of it.
Combination of those two mechanism together makes online communication becomes much more trustworthy.
# REAL LIFE SCENARIO

Let us examine this example:

- **John**: Hey Robert, can you send those confidential data to Ashley?
- **Robert**: Sure, but who is she?
- **John**: She is the HR manager of BBSec.
- **Robert**: All right.

<br>For Robert to send any confidential data to Ashley, he must find Ashley's public-key. However, searching for her
public-key on the web would not be smart since there could be many other public-keys for other Ashleys. How can Robert
be confident that he uses the correct public-key from the Ashley he wants to communicate?
<br><br>To solve this problem, what we call **_x509 certificates_** comes into play.

# WHAT IS AN X.509 CERTIFICATE?

_X.509_ is a standard procedure for public-key certificates. They are digital documents that validate the integrity of the 
association between identities such as organizations, individuals or websites, and their cryptographic key pairs.
<br><br>An _X.509 certificate_ essentially contains those:
- Identity's name, address, email,etc.
- Identity's public-key.
- The _Issuer_ of the certificate.
- The _encrypted hash_ of the certificate.

If Robert can find a certificate that binds Ashley's name with her company BBSec, he can confidently tell that it is the correct
certificate he was looking for.
<br><br>There are two more fields above, which are _Issuer_ and _encrypted hash_. We will come to that later.

Common applications that use X.509 certificates:
- TLS/SSL and HTTPS for encrypted and authenticated web browsing
- S/MIME Protocol for encrypted and signed email. 
- Document signing
- Client authentication
- Goverment-issued electronic ID

# CERTIFICATE AUTHORITIES

However, what if Robert finds more than one certificate claiming to have Ashley's public key? Now which one is the right one?
I can easily create a certificate identical to Ashley's and put my public-key in it.
We need a mechanism to verify that the certificate assigned to a specific identity is one of its kind.
<br><br>We have what we call **_Certificate Authorities (CAs)_** to solve this problem.  There are dozens of organizations that we 
trust to be authoritative. All operating systems and browsers come with built-in configured CAs. If a CA claims
a certificate is authentic, we will believe it. 
<br><br>However, how does a CA indicate that the certificate in question is authentic?
By _signing_ it digitally. The signature of the CA must be checked to verify the authenticity of a certificate.
<br><br> In the definition of an X.509 certificate above, there were two more fields left unexplained. _Issuer_ and _Encrypted Hash_.
- Issuer: The name of the organization (CA) that issued the certificate.
- Encrypted Hash: the digital signature of the CA.

Let us continue to the Robert and Ashley example above:

1. Robert obtains certificate A that claims to be Ashley's certificate and issued by X.
2. To verify that certificate A is authentic by verifying CA's signature, Robert gets the public key of X from somewhere, then
decrypts the _encrypted hash_, the digital signature, inside of certificate A to see if it matches. 

The question is that Robert just downloaded X's certificate from somewhere, so how can he tell that it is also authentic? By looking
at the _Issuer_ of X, which is Y. Now, he must find the Y certificate to verify the signature on the X certificate. We can continue
to do that endlessly. It becomes an endless loop, and we have to stop at one point.
<br><br>To solve this problem we have what we call the **_Root CAs_**.

# WHAT IS A ROOT CA?

A **_top-level, root CA_** is a certificate authority that sits on the top of the authority hierarchy. CAs can sign their own certificates. These
are what we call _self-signed certificates_. Since it is not possible to validate self-signed certificates with PKI, Root CAs 
distribute their keys _out-of-band_ (e.g. out-of-band distribution means non-electronic distributing, such as going to their offices
and getting their public key in person).
<br><br>However, it would be harsh to go to the Root CAs office and collect their public key each time you intend to send an 
email to somebody. Web browsers come with more than 30 Root CAs. The browsers' distributors are deciding who are truthful. By
using their browser, you trust their choice.
<br><br>Let us get back to the Ashley and Robert example. If we assume that the certificate Y is a _self-signed_ certificate and the organization Y
is indeed a _Root CA_, Robert can trust certificate A since X trusts A and Y trusts X.

# TERMINOLOGIES

## X.509
The protocol's itself.
## PKIX
Stands for **P**ublic  **K**ey **I**nfrastructure **X**.509
## [Abstract Syntax Notation One (ASN.1)](https://en.wikipedia.org/wiki/ASN.1)
It is the syntax used to describe things in an X.509 certificate. For instancee, if the certificates were written in JSON, 
ASN.1 would be the syntax for the schema (This example is over-simplified).
Using ASN.1, it is possible to define data structures that 
can be transmitted over the network independently of software or hardware.
### DER, PEM, XER, PER, BER, CER
Those are some of the popular encoding rules for ASN.1. 
<br>_DER_ and _PEM_ are the used.
## Algorithm
When the context is the PKI, it refers to cryptographic algorithms like "SHA1 with DSA", "SHA1 with RSA", etc. In order to
achieve cryptographic signing, first, we need to hash the data, then encrypt the hash. "SHA1 with RSA" would indicate that the
data hashed with SHA1 and encrypted with RSA.
## [Object Identifiers (OIDs)](https://en.wikipedia.org/wiki/Object_identifier)
Instead of using English to describe things in a X.509 certificate, _OIDs_ are being used.
For instance, in a certificate 
instead of referring to the encryption field as "RSA", which would be a string value, you would see its' integer OID value "1.2.840.11359".
<br>[You can find the list of the registered OIDs here.](https://www.alvestrand.no/objectid/top.html) 
## [Privacy Enhanced Mail (PEM)](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail)
If you hear PEM, it usually refers to [DER](https://asecuritysite.com/encryption/sigs3?a0=304e301006072a8648ce3d020106052b81040021033a0004eada93be10b2449e1e8bb58305d52008013c57107c1a20a317a6cba7eca672340c03d1d2e09663286691df55069fa25490c9dd9f9c0bb2b5). 
PEM is further encoded version of DER to base64.
Using PEM encoding can assist transmission over media that is sensitive to textual encodings.
## [Distinguished Name (DN)](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/ldap/distinguished-names?redirectedfrom=MSDN)
Just referring someone with his/her name would not be helpful to identify someone. 
_DN_ is used to identify someone uniquely by dividing attributes into fields like **CN=Burak, DC=bbsec, DC=COM**.
## [Simple PKI (SPKI)](https://crypto.stackexchange.com/questions/790/need-an-introduction-to-spki-or-spki-for-dummies)
The PKI we were discussing ([RFC 5280](https://www.ietf.org/rfc/rfc5280.txt)) binds a certificate with _distinguished name_.
SPKI ([RFC 2692](https://datatracker.ietf.org/doc/html/rfc2692)) binds public key to a set of permissions. Not commonly used.

## [Public Key Cryptography Standrat (PKCS)](https://www.encryptionconsulting.com/public-key-cryptography-standards/)
_PKCS_ is a set of standrats for PKI from 1 to 15. The PKCS standarts listed below are the most common and important ones.

- PKCS #1: RSA Cryptograpgy Standart
- PKCS #3: Diffie-Hellman Key Agreement Standard
- PKCS #7/CMS: Cryptographic Message Syntax Standard 
- PKCS #9: Selected Object Classes and Attribute Types
- PKCS #10: Certification Request Syntax Standard
- PKCS #12: Personal Information Exchange Syntax Standard

## [Certificate Revocation List (CLR)](https://en.wikipedia.org/wiki/Certificate_revocation_list)
These lists are published by some authorities' servers to indicate which certificates have been revoked.
If you were to lose your credit card for whatever reason, you would like to revocate your credit card.
It works the same for the certificates too. You might lose your private key for whatever reason, or it can get stolen. In this case, 
you would like to revocate your X.509 certificate.

## [Online Certificate Status Protocol (OCSP)](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol)
An internet protocol used for gathering the revocation status of an X.509 certificate.

## [Certificate Signing Requests (CSR)](https://www.namecheap.com/support/knowledgebase/article.aspx/337/67/what-is-a-certificate-signing-request-csr/)
In order to get a certificate signed by a Root CA, a _Certificate Signing Request_ should be made to a Root CA.
There are meta-data that needs to be bundled which is specified by PKCS#10

