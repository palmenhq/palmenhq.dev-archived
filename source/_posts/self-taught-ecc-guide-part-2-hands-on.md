---
title: The self-taught Elliptic Curve Cryptography Guide - Part 2 (Hands-on)
date: 2022-02-04 12:24:33
tags:
    - cryptography
    - explained
---

For the last few months I’ve spent some time learning about crypto (not the hype kind of crypto, just OG cryptography). In this blog series I will try to summarize my learnings so far. I’m far from an expert, especially when it comes to the math behind it all, but I figured I’ve learnt a lot and want to share those learnings. But please; use a well-known library (such as OpenSSL) for your crypto implementations - never roll your own crypto. With that said, knowing some theory behind it will help you greatly when using the libraries.

The {% post_link "self-taught-ecc-guide-part-1-what-it-is" "first part" %} will mostly cover the theory that will enable you to actually understand what’s happening in this second, more hands-on, part. I do recommend spending time with the theory before diving into the practical stuff, but feel free to skip ahead if you know some concepts already.

## Key Formats

There are different key formats and encodings for cryptographic keys and other artifacts, I will try to summarize a couple of common ones here.

### ASN.1 and DER

Most cryptographic assets (keys, certificates etc.) build on ASN.1, which is a way to describe how to (de)serialize data (think JSON-schema). The most common format for cryptographic assets is DER. An example of an ANS.1 module (data description) and example serialization using DER might look something like this:

```asn1
Point ::= SEQUENCE {
  x INTEGER,
  y INTEGER,
  label UTF8String
}
```

In TypeScript the same definition would translate into

```ts
type Point = [x: number, y: number, label: string]
```

An example of a binary DER encoded Point is;

```c
30 10 02 01 09 02 01 70 12 08 6d 79 20 70 6f 69 6e 74
```

The value can be described as the following

```txt
SEQUENCE (3 elem)
  INTEGER 9
  INTEGER 112
  NumericString "my point" (NumericString aka UTF8 string)
```

In TypeScript, it would look something like this:

```ts
[9, 112, "my point"]
```

Let's break down the hex bytes:

```c
30 10 // here comes a sequence (type 30) consisting of 16 bytes (10 in hex)
    02 01 // 1st item, an integer (data type 02) of 1 byte
        09 // value 9
    02 01 // 2nd item, an integer of 1 byte
        70 // value 112 in hex
    12 08 // 3rd item, a utf8 string (12) of 8 bytes
        6d 79 20 70 6f 69 6e 74 // "my point" in hex    
```

A very good tool for debugging ASN.1 is the [ASN.1 JavaScript decoder](https://lapo.it/asn1js/), which does not only a great job of decoding ASN.1 formats in a readable way, it also interactively visualizes the hex dump so the DER encoding becomes more readable. I do recommend pasting the above example hex (or using this [direct link](https://lapo.it/asn1js/#MBACAQkCAXASCG15IHBvaW50)) and familiarizing yourself with the ASN.1 debugger if you plan on working with "raw" crypto assets.

### PEM

PEM is an attempt to make DER a little more comprehensible by encoding the DER as base64 (instead of raw binary) and labeling its contents (i.e. `PUBLIC KEY` or `PRIVATE KEY` etc). An example private key PEM (represented as a SEC1 EC Key, see below) might look something like this;

```pem
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIJkK9PKZtn/Q27/h9Msl2ThBBPT/7+4Z+HTH7aZl0MEpoAoGCCqGSM49
AwEHoUQDQgAEofYx1+G2JhjVczsvvPp7JxemaSKKle0KXG+JHMKy2W0Hy5xcrfhG
Ai1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END EC PRIVATE KEY-----
```

As you can see, it starts and ends with a label (`BEGIN|END EC PRIVATE KEY`), and has some base64 gibberish (not actually gibberish) between. Let's introspect a couple of PEM files. 

### The SEC1 EC Key Format

A private key can be represented in multiple forms but the variant described in SEC 1 (the paper can be found at [Standards for Efficient Cryptography Group's website](http://www.secg.org/)) is common (and the default output OpenSSL uses). Let's introspect the above private key using the ASN.1 debugger (paste it or use [this link](https://lapo.it/asn1js/#MHcCAQEEIJkK9PKZtn_Q27_h9Msl2ThBBPT_7-4Z-HTH7aZl0MEpoAoGCCqGSM49AwEHoUQDQgAEofYx1-G2JhjVczsvvPp7JxemaSKKle0KXG-JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4-UO3HvGMKqY7AA));  

```txt
SEQUENCE (4 elem)
  // version:
  INTEGER 1
  // the actual key private key:
  OCTET STRING (32 byte) 990AF4F299B67FD0DBBFE1F4CB25D9384104F4FFEFEE19F874C7EDA665D0C129
  [0] (1 elem)
    // description of the public key (this key is a NIST P-256 aka prive256v1)
    OBJECT IDENTIFIER 1.2.840.10045.3.1.7 prime256v1 (ANSI X9.62 named elliptic curve)
  [1] (1 elem)
    // the actual public key in bits
    BIT STRING (520 bit) 0000010010100001111101100011000111010111111000011011011000100110000110…
```

Another common standard is [PKCS#8](https://en.wikipedia.org/wiki/PKCS_8) (PKCS is a set of numbered standards, so #8 doesn't mean version 8, it's a completely different standard to i.e. PKCS#12, took me a while to realize), but I will scope that out to keep these blog posts less huge.

### SPKI

[Decoding the PEM](https://lapo.it/asn1js/#MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEofYx1-G2JhjVczsvvPp7JxemaSKKle0KXG-JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4-UO3HvGMKqY7AA) of the corresponding public key, we'll get something looking completely different, because it is encoded in SPKI;

```pem
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEofYx1+G2JhjVczsvvPp7JxemaSKK
le0KXG+JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END PUBLIC KEY-----
```

(This can of course also be described in hex:)

```
30570201010420990af4f299b67fd0dbbfe1f4cb25d9384104f4ffefee19
f874c7eda665d0c129a00a06082a8648ce3d030107a12403220002a1f631
d7e1b62618d5733b2fbcfa7b2717a669228a95ed0a5c6f891cc2b2d96d
```

ANS.1 decoder result:

```pem
SEQUENCE (2 elem)
  SEQUENCE (2 elem)
    OBJECT IDENTIFIER 1.2.840.10045.2.1 ecPublicKey (ANSI X9.62 public key type)
    OBJECT IDENTIFIER 1.2.840.10045.3.1.7 prime256v1 (ANSI X9.62 named elliptic curve)
  BIT STRING (520 bit) 0000010010100001111101100011000111010111111000011011011000100110000110…
```

In the case of a NIST P-256 curve the bit string at the end is always also on octet string (but it can also be an "uneven" number of bits for other curves), and can thus be encoded as the following 65-byte (520/8) hex-string:

```txt
04a1f631d7e1b62618d5733b2fbcfa7b2717a669228a95ed0a5c6f891cc2
b2d96d07cb9c5cadf846022d4a4a1725d24102f6112ab9f378f943b71ef1
8c2aa63b00
```

The above hex is a raw and uncompressed public key. It's divided in three parts;

1. The first byte, `04`, indicates that this is indeed uncompressed (if a public key starts with `02` that means it's compressed, and the two coordinates are combined into 32 bytes)
2. The next 32 bytes is the x-coordinate on the elliptic curve
3. The 32 bytes after that is the y-coordinate on the elliptic curve

## NIST P-256 and OpenSSL

Let's dive into some practicalities of how to work with EC keys in OpenSSL

### Generating a Private Key

Use `openssl ecparam -genkey` to generate the NIST P-256 (aka prime256v1, remember?) private key and store it to the file `mykey.pem`

```sh
$ openssl ecparam -genkey -out mykey.pem -name prime256v1
```

### Inspect the private key

First let's take a look at the "raw" pem file (same as under [PEM](#PEM) except it also includes some ec parameters).

```sh
$ cat mykey.pem
-----BEGIN EC PARAMETERS-----
BggqhkjOPQMBBw==
-----END EC PARAMETERS-----
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIJkK9PKZtn/Q27/h9Msl2ThBBPT/7+4Z+HTH7aZl0MEpoAoGCCqGSM49
AwEHoUQDQgAEofYx1+G2JhjVczsvvPp7JxemaSKKle0KXG+JHMKy2W0Hy5xcrfhG
Ai1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END EC PRIVATE KEY-----
```

Here's how we will inspect the key. `-text` shows us the data in human-readable form (so we don't only get the pem). As you can see it has the same data as the decoded [SEC1 EC Key DER](#The-SEC1-EC-Key-Format) (it's just displayed differently).

```sh
$ openssl ec -text -in mykey.pem
read EC key
Private-Key: (256 bit)
priv:
    00:99:0a:f4:f2:99:b6:7f:d0:db:bf:e1:f4:cb:25:
    d9:38:41:04:f4:ff:ef:ee:19:f8:74:c7:ed:a6:65:
    d0:c1:29
pub:
    04:a1:f6:31:d7:e1:b6:26:18:d5:73:3b:2f:bc:fa:
    7b:27:17:a6:69:22:8a:95:ed:0a:5c:6f:89:1c:c2:
    b2:d9:6d:07:cb:9c:5c:ad:f8:46:02:2d:4a:4a:17:
    25:d2:41:02:f6:11:2a:b9:f3:78:f9:43:b7:1e:f1:
    8c:2a:a6:3b:00
ASN1 OID: prime256v1
NIST CURVE: P-256
writing EC key
-----BEGIN EC PRIVATE KEY-----
MHcCAQEEIJkK9PKZtn/Q27/h9Msl2ThBBPT/7+4Z+HTH7aZl0MEpoAoGCCqGSM49
AwEHoUQDQgAEofYx1+G2JhjVczsvvPp7JxemaSKKle0KXG+JHMKy2W0Hy5xcrfhG
Ai1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END EC PRIVATE KEY-----
```

As you can see, our private key here is

```
00990af4f299b67fd0dbbfe1f4cb25d9384104f4ffefee19f874c7eda665d0c129
```

and our (raw uncompressed) public key is embedded as

```
04a1f631d7e1b62618d5733b2fbcfa7b2717a669228a95ed0a5c6f891cc2
b2d96d07cb9c5cadf846022d4a4a1725d24102f6112ab9f378f943b71ef1
8c2aa63b00
```

### Deriving a Public Key Pem

To derive the public key pem, we can add `-pubout` when we inspect the private key. Again, use `-text` to get the details in human-readable form.

```sh
# Derive (and inspect) the public key
$ openssl ec -text -in mykey.pem -pubout
read EC key
Private-Key: (256 bit)
priv:
    00:99:0a:f4:f2:99:b6:7f:d0:db:bf:e1:f4:cb:25:
    d9:38:41:04:f4:ff:ef:ee:19:f8:74:c7:ed:a6:65:
    d0:c1:29
pub:
    04:a1:f6:31:d7:e1:b6:26:18:d5:73:3b:2f:bc:fa:
    7b:27:17:a6:69:22:8a:95:ed:0a:5c:6f:89:1c:c2:
    b2:d9:6d:07:cb:9c:5c:ad:f8:46:02:2d:4a:4a:17:
    25:d2:41:02:f6:11:2a:b9:f3:78:f9:43:b7:1e:f1:
    8c:2a:a6:3b:00
ASN1 OID: prime256v1
NIST CURVE: P-256
writing EC key
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEofYx1+G2JhjVczsvvPp7JxemaSKK
le0KXG+JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END PUBLIC KEY-----
```

To derive a public key pem file, (sometimes suffixed with `.pub`) you can omit `-text` and just use `-out` to write the resulting pem;

```sh
$ openssl ec -in mykey.pem -pubout -out mykey.pub
read EC key
writing EC key
$ cat mykey.pub
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEofYx1+G2JhjVczsvvPp7JxemaSKK
le0KXG+JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END PUBLIC KEY-----
```

We can now introspect the public key by using `-pubin`:

```sh
$ openssl ec -pubin -in mykey.pub -text
read EC key
Private-Key: (256 bit)
pub:
04:a1:f6:31:d7:e1:b6:26:18:d5:73:3b:2f:bc:fa:
7b:27:17:a6:69:22:8a:95:ed:0a:5c:6f:89:1c:c2:
b2:d9:6d:07:cb:9c:5c:ad:f8:46:02:2d:4a:4a:17:
25:d2:41:02:f6:11:2a:b9:f3:78:f9:43:b7:1e:f1:
8c:2a:a6:3b:00
ASN1 OID: prime256v1
NIST CURVE: P-256
writing EC key
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEofYx1+G2JhjVczsvvPp7JxemaSKK
le0KXG+JHMKy2W0Hy5xcrfhGAi1KShcl0kEC9hEqufN4+UO3HvGMKqY7AA==
-----END PUBLIC KEY-----
```

## Key Fingerprinting

A common way of identifying key pairs is by using key fingerprinting. A key fingerprint is commonly just a hash (or part of a hash) of the public key. However, when you do make a fingerprint of a key, it is extremely important that you always use the same (expected) form of public key. As you should understand by now, performing `sha2(public key pem)` will yield something completely different to performing `sha2(raw uncompressed public key)`, just like `sha2(spki-encoded public key)` will be equally different. In addition, `sha2(hex encoded uncompressed public key)` will be completely different form `sha2(raw uncompressed public key)`.

A common application of key fingerprinting is within the crypto-currency community. For example, Ethereum addresses are just public key fingerprints. They are derived from the public key by using the last 20 bytes of a hash of the raw uncompressed form of the public key (without the leading `04` byte). It's not an optimal example because Ethereum uses curve secp256k1 public keys (instead of secp256v1 that I've described so far, but from what I've understood they are very similar), and Keccak-256 hashing (that later evolved into SHA3, but they are not the same), while I've described SHA2. However, the principle is exactly the same. An Ethereum address can thus be derived from its private key by using; `last_20_bytes(Keccak256(raw public key))`. Here's an example (you can try it by using an [online keccak-256 hasher](https://emn178.github.io/online-tools/keccak_256.html));

```text
Raw uncompressed public key:
048e66b3e549818ea2cb354fb70749f6c8de8fa484f7530fc447d5fe80a1
c424e4f5ae648d648c980ae7095d1efad87161d83886ca4b6c498ac22a93
da5099014a

Hashing material is thus (exactly the same as above but without leading 04):
8e66b3e549818ea2cb354fb70749f6c8de8fa484f7530fc447d5fe80a1c4
24e4f5ae648d648c980ae7095d1efad87161d83886ca4b6c498ac22a93da
5099014a

Using Keccak-256 to hash the byte string (not the hex string) we get:
89c65437e880d5c3880d724000b54e93ee2eba3086a55f4249873e291d1ab06c

Taking the last 20 bytes (40 hex characters) gives us the address (prefixed with `0x`):
0x00b54e93ee2eba3086a55f4249873e291d1ab06c
```

You can try it yourself by using an [Ethereum key-to-address converter](https://toolkit.abdk.consulting/ethereum#recover-address,key-to-address) Commonly Ethereum hex strings are prefixed with `0x`, but as you know this is not part of the hex value.

## Conclusion

There's many moving parts in cryptography, and I'd recommend doing as little as possible yourself, but with this knowledge you should at least know a little more about the libraries you use. They all do the same things ultimately, and mostly it is OpenSSL doing the real work under the hood.

There are some essential parts to ECC that I've purposely omitted because it's just too much to cover in a couple of blog posts. But expect a follow-up describing signing/verifying content with ECDSA and maybe something on symmetric cryptography and Diffie-Hellman Key Exchange (DH/ECDH).

If you found this useful, or not (or if you found any issues), please do let me know on [Twitter](https://twitter.com/palmenhq)! Congratulations and thanks for making it all the way 🌞
