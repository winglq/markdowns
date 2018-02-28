title: Crypto
author: qing
date: 2018-02-28
description: Things about encrypting and signing ...
tags:
category:

# Crypto

## Encrypting
When encrypting, you use their public key to write message and they use their private key to read it.

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
*In a public key encryption system, any person can encrypt a message using the receiver's public key.* That encrypted message can only be decrypted with the receiver's private key. To be practical, the generation of a public and private key -pair must be computationally economical. 

## Signing
When signing, you use your private key to write message's signature, and they use your public key to check if it's really yours.
A signing and verifying example.

    # You can create a detached signature for a message using the rsa.sign() function:
    >>> (pubkey, privkey) = rsa.newkeys(512)
    >>> message = 'Go left at the blue tree'
    >>> signature = rsa.sign(message, privkey, 'SHA-1')
    # This hashes the message using SHA-1. Other hash methods are also possible, check the rsa.sign() function documentation for details. The hash is then signed with the private key.
    # In order to verify the signature, use the rsa.verify() function. This function returns True if the verification is successful
    >>> message = 'Go left at the blue tree'
    >>> rsa.verify(message, signature, pubkey)
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
