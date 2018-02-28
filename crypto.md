title: Crypto
author: qing
date: 2018-02-28
description: Things about encrypting and signing ...
tags:
category:

# Crypto

---

## Crypting
**When encrypting, you use their public key to write message and they use their private key to read it.**

A sample code which demotrates how to ecncrypt/decrypt using private key only.(symmetric)

    from Crypto.Cipher import AES
    import base64
    
    msg_text = 'test some plain text here'.rjust(32)
    secret_key = '1234567890123456' # private key. create new & store somewhere safe. 16 Bytes 128 bits => AES-128
    
    cipher = AES.new(secret_key,AES.MODE_ECB) # never use ECB in strong systems obviously
    encoded = base64.b64encode(cipher.encrypt(msg_text))
    # ...
    # another size who received the encrypted data
    decoded = cipher.decrypt(base64.b64decode(encoded))
    print decoded.strip()
    
The next sample code demotrates an asymmetrical cryptography. To run the following code you need to install rsa package.

    # 1. Bob generates a keypair, and gives the public key to Alice. This is done such that Alice knows for sure that the key is really Bob’s (for example by handing over a USB stick that contains the key).
    >>> import rsa
    >>> (bob_pub, bob_priv) = rsa.newkeys(512)
    # 2. Alice writes a message, and encodes it in UTF-8. The RSA module only operates on bytes, and not on strings, so this step is necessary.
    >>> message = 'hello Bob!'.encode('utf8')
    # 3. Alice encrypts the message using Bob’s public key, and sends the encrypted message.
    >>> import rsa
    >>> crypto = rsa.encrypt(message, bob_pub)
    # 4. Bob receives the message, and decrypts it with his private key.
    >>> message = rsa.decrypt(crypto, bob_priv)
    >>> print(message.decode('utf8'))
    hello Bob!
    
### AES
AES `(Advanced Encryption Standard)` is a symmetric block cipher standardized by NIST_ . It has a fixed data block size of 16 bytes. Its keys can be 128, 192, or 256 bits long.
AES is very fast and secure, and it is the de facto standard for symmetric encryption.

### Base64 (not a encrypt method but a encode method)
Basically Base64 converts a string of bytes into a string of ASCII characters so that they can be safely transmitted within HTTP headers.
A byte consist of 8 bits. When encoding, Base64 will divide the string of bytes into groups of 6 bits and each group will map to one of 64 characters. These 64 characters are the in the Base64 character table.
For example: 0 -> A, 1 -> B, ..., 25 -> Z, 26 -> a, ..., 51 -> z, ..., 63 -> /

### RSA
RSA is an asymmetric system , which means that a key pair will be generated (we will see how soon) , a public key and a private key , obviously you keep your private key secure and pass around the public one.
RSA is rather slow so it’s hardly used to encrypt data , more frequently it is used to encrypt and pass around symmetric keys which can actually deal with encryption at a faster speed.

### Public key cryptography, or asymmetrical cryptography
Public key cryptography, or asymmetrical cryptography, is any cryptographic system that uses pairs of keys: public keys which may be disseminated widely, and private keys which are known only to the owner. This accomplishes two functions: authentication, where the public key verifies a holder of the paired private key sent the message, and encryption, where only the paired private key holder can decrypt the message encrypted with the public key.
**In a public key encryption system, any person can encrypt a message using the receiver's public key.** That encrypted message can only be decrypted with the receiver's private key. To be practical, the generation of a public and private key -pair must be computationally economical.

## Signing
**When signing, you use your private key to write message's signature, and they use your public key to check if it's really yours.**
A signing and verifying example.

    # You can create a detached signature for a message using the rsa.sign() function:
    >>> (pubkey, privkey) = rsa.newkeys(512)
    >>> message = 'Go left at the blue tree'
    >>> signature = rsa.sign(message, privkey, 'SHA-1') # use privkey to crypt message and use SHA-1 to get the digest data.
    # This hashes the message using SHA-1. Other hash methods are also possible, check the rsa.sign() function documentation for details. The hash is then signed with the private key.
    # In order to verify the signature, use the rsa.verify() function. This function returns True if the verification is successful
    >>> message = 'Go left at the blue tree'
    >>> rsa.verify(message, signature, pubkey) # use pubkey to crypt message and calculate SHA-1 digest and compare the result to signature.
    True
    
### SHA-1
In cryptography, SHA-1 (Secure Hash Algorithm 1) is a cryptographic hash function which takes an input and produces a 160-bit (20-byte) hash value known as a message digest - typically rendered as a hexadecimal number, 40 digits long. It was designed by the United States National Security Agency, and is a U.S. Federal Information Processing Standard.
SHA-1 example.

    import hashlib
    m = hashlib.sha1()
    m.update("Nobody inspects")
    m.update(" the spammish repetition")
    print m.digest()
    print m.digest_size
    
### Sign CSR procedure

* Application owner generates key pair(public/private key)
* Create CSR including organization information(such as a distinguished name in the case of an X.509 certificate) and public key
* Sign the information using private key(CA can use signature and public key to verify the infomation)
* Send the certificate request file to the CA (Certification Authority) for signing and save the signed certificate request file on your local machine.

### Client check certificate procedure (this is a part of the SSL/TSL procedure)

* Client receives server's certificate
* Client uses CA's public key to verify that the certificate is never modified.
* Client uses the public key in the certificate to encrypt information.
* Server decrypt infomation by private key.
* new key change
    
### X.509
In cryptography, X.509 is a standard that defines the format of public key certificates. X.509 certificates are used in many Internet protocols, including TLS/SSL, which is the basis for HTTPS, the secure protocol for browsing the web. They are also used in offline applications, like electronic signatures.

Certificate

  * Version Number
  * Serial Number
  * Signature Algorithm ID
  * Issuer Name
  * Validity period
    * Not Before
    * Not After
  * Subject name
  * Subject Public Key Info
    * Public Key Algorithm
    * Subject Public Key
  * Issuer Unique Identifier (optional)
  * Subject Unique Identifier (optional)
  * Extensions (optional)
    * ...
  * Certificate Signature Algorithm
  * Certificate Signature
  
## TLS/SSL
SSL (and its successor, TLS) is a protocol that operates directly on top of TCP (although there are also implementations for datagram based protocols such as UDP). This way, protocols on higher layers (such as HTTP) can be left unchanged while still providing a secure connection. Underneath the SSL layer, HTTP is identical to HTTPS.
Working flow:

* TCP connect 
* SSL handshake (SSL version/cipher suite/compression method) 
* server send certificate to client(signed by authority)
* verify the certificate and being certain this server really is who he claims to be (and not a man in the middle).
* change a new key. This can be a public key, a "PreMasterSecret" or simply nothing, depending on the chosen ciphersuite. Both the server and the client can now compute the key for the symmetric encryption.
* The client tells the server that from now on, all communication will be encrypted, and sends an encrypted and authenticated message to the server.
* The server verifies that the MAC (used for authentication) is correct, and that the message can be correctly decrypted. It then returns a message, which the client verifies as well.
* The handshake is now finished, and the two hosts can communicate securely.

### Server Name Indication
Server Name Indication (SNI) is an extension to the TLS computer networking protocol by which a client indicates which hostname it is attempting to connect to at the start of the handshaking process. This allows a server to present multiple certificates on the same IP address and TCP port number and hence allows multiple secure (HTTPS) websites (or any other Service over TLS) to be served by the same IP address without requiring all those sites to use the same certificate. It is the conceptual equivalent to HTTP/1.1 name-based virtual hosting, but for HTTPS. The desired hostname is not encrypted, so an eavesdropper can see which site is being requested.

TLS handshake happens before the server sees any HTTP headers. Therefore, it is not possible for the server to use the information in the HTTP host header to decide which certificate to present and as such only names covered by the same certificate can be served from the same IP address.
**SNI addresses this issue by having the client send the name of the virtual domain as part of the TLS negotiation.**
