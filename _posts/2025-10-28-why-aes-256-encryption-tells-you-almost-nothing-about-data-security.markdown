---
layout: post
title: "Why 'AES-256 Encryption' Tells You Almost Nothing About Data Security"
date: 2025-10-28 17:00:0 +1000
---

It's common to see companies attesting that their customer's data is secure because they use "AES-256" encryption. Some might go as far as saying that they use "military-grade" encryption, which conveys absolutely no information because it is not a well-defined term. Which military? The term "AES-256" is better defined, but is still not precise enough. The at-rest encryption of data is also only a small component of a service's security posture.

## Symmetric Encryption

Symmetric encryption is a type of encryption that uses the same key to encrypt and decrypt the data. This is in contrast to public-key cryptography (asymmetric encryption) where a pair of keys can be derived such that one can be used for encryption (the public key) and another for decryption (the private key). Despite this useful property, asymmetric encryption is generally slower and provides less defence against brute-force attack for keys of the same size as an equivalent symmetric algorithm

For these reasons, symmetric encryption is _usually_ preferred for at-rest encryption of data (e.g. data volumes or backups). A single private key is used for both encryption and decryption.

## Block Ciphers

Far and above the most common set of ciphers (algorithms) used for at-rest encryption come from the Advanced Encryption Standard (AES), which was defined by the US National Institute of Standards and Technology (NIST) in 2001. NIST held a competition and invited cryptographers to submit their algorithms. The winning submission was the Rijndael block cipher, submitted by a pair of Belgian researchers, Vincent **Rij**men and Joan **Dae**man.

The AES set of ciphers are a form of block cipher; a cipher that operates on data of fixed size. For AES, this block size is 128 bits. So where does the _-256_ in AES-256 come from? That refers to the size of the key used; 256 bit keys are used for **AES-256**.

This property of block ciphers presents an immediate challenge. If the cipher only supports block sizes of 128 bits, how is the algorithm used to encrypt data that is many orders of magnitude larger? Naturally, the answer is to loop over the data and apply the cipher repeatedly in 128bit chunks until the entire payload has been encrypted. The simplest (naive) approach is something like:

```python
encrypted = []
for chunk in plaintext:
 encrypted += AES256(privatekey, chunk)
```

This loop, describing how a block cipher is applied in chunks, is referred to as a _mode of operation_. This particular mode of operation is known as Electronic Codebook (ECB) mode. If you're not familiar with block ciphers, this approach might seem reasonable, but despite the strength of the AES256 cipher the overall implementation is terribly insecure. This mode of operation introduces a property that two chunks with the same plaintext will have the same encrypted output. A great example of why this is a bad idea is provided below ([source]). When looking at an individual encrypted pixel, it's impossible to tell what the input is. But zooming out and seeing the image as a whole reveals very clear patterns in the input data

![ECB Penguin](/images/An-example-of-image-encrypted-using-ECB-mode.png)

So when a service says they are using "AES-256", it's possible that they are using AES-256-ECB. While unlikely, it was discovered back in 2020 that while Zoom was claiming to use "AES-256" to encrypt their meetings what they were actually doing was:

- Generating a single AES-128 key for each meeting on Zoom servers (these servers often resided in countries completely seperate from the meeting participants)
- Supplying that key to every meeting participant
- Encrypting all traffic with AES-128 in **Electronic Codebook Mode (!!!)**

So while ECB is generally regarded as a terrible idea, it didn't stop Zoom from using this scheme. There is a great [blog post](https://citizenlab.ca/2020/04/move-fast-roll-your-own-crypto-a-quick-look-at-the-confidentiality-of-zoom-meetings/) that describes the shortcomings of Zoom's cryptography, if you're interested in learning more.

What's the solution? There are much better modes of operation available. One classic method is known as Cipher Block Chaining (CBC). In this mode, a random (but not secret) initialisation vector (IV) is generated and XORd with the first block prior to encryption. For all subsequent blocks, the previous encrypted chunk is XORd with the current plaintext chunk prior to encryption. It looks something like this:

```python
encrypted = []
previous_block = IV # Initialization Vector (random, same size as block)

for chunk in plaintext:
 xor_result = XOR(chunk, previous_block)
 cipher_block = AES256(privatekey, xor_result)
 encrypted += cipher_block
 previous_block = cipher_block
```

This has the property that chunks with the same content will look totally different when encrypted. However, it's critical that the IV used is unique and unpredictable. Older methods of SSL which used CBC were [vulnerable](https://openssl-library.org/files/tls-cbc.txt) to these sorts of attacks, because the chosen IV was predictable.

One of the most common modes you will see today is Galois Counter Mode (GCM) which takes a different approach, but provides similar protections against pattern identification in ciphertext.

As an aside, be extremely careful using Universally Unique Identifiers (UUIDs) in cases where you don't want an attacker being able to guess the ids. For example, storing user content behind urls (with no additional access control) with the assumption that users will not be able to guess these URLs. Per the original [RFC]():

> **Do not assume that UUIDs are hard to guess**; they should not be used
> as security capabilities (identifiers whose mere possession grants
> access), for example. A predictable random number source will
> exacerbate the situation.

It seems possible to implement unpredictable UUIDs in newer versions of the standard, but this also depends on the implementation. Best to avoid them entirely in cases where unpredictability is important.

## Data Protection

While I've only scratched the surface of symmetric encryption, it should now be clear that while AES-256 is a great block cipher, being told that "your data is protected with AES-256" shouldn't give you much comfort. There are many ways to make the encryption vulnerable through selection (and implementation) of operation mode. This is why the standard advice is to _"never roll your own crypto"_, preferring use of standard libraries like OpenSSL instead.

Beyond shoddy crypto implementations, there is a famous [quote](https://amturing.acm.org/vp/shamir_2327856.cfm) from Adi Shamir (the "S" in RSA encryption) stating that:

> Cryptography is typically bypassed, not penetrated

It's rare that weak cryptography is the basis for a data breach. There have been countless breaches resulting from misconfigured permissions on S3 buckets, this is despite the fact that buckets use AES-256-GCM encryption by default. No systems or cryptographic ciphers worked unexpectedly, it's just that the access control implemented was fundamentally broken.

So while it's great that your service encrypts my data using the AES-256 block cipher, it really tells me nothing about the security of your systems.
