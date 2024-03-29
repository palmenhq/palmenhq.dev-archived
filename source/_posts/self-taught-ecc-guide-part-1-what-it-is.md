---
title: The self-taught Elliptic Curve Cryptography Guide - Part 1 (What it is)
date: 2022-03-11 11:24:32
tags:
    - cryptography
    - explained
---

_For the last few months I’ve spent some time learning about crypto (not the hype kind of crypto, just OG cryptography). In this blog series I will try to summarize my learnings so far. I’m far from an expert, especially when it comes to the math behind it all, but I figured I’ve learnt a lot and want to share those learnings. But please; use a well-known library (such as OpenSSL) for your crypto implementations - never roll your own crypto. With that said, knowing some theory behind it will help you greatly when using the libraries._

_The first part will mostly cover the theory that will enable you to actually understand what’s happening in {% post_link "self-taught-ecc-guide-part-2-hands-on" "the second" %}, more hands-on, part. I do recommend spending time with the theory before diving into the practical stuff, but feel free to skip ahead if you know some concepts already._

## Binary-formats Memory Refresh (don't skip me please)

If you have a degree in computer science (unlike me) I'm sure you know binary formats and representations well. However, I really think it's good to go through a memory refresh, as keeping track of formats and encodings is critical in the context of crypto. Don't skip this part, if you know it - just feel good about how much you know and make sure you don't miss anything :)

The important takeaway here is that as every value in a computer is represented as one or more bits, it can be represented as a number. Thus, we can make mathematical calculations (thus cryptographic operations) on any value (even a picture or text string).

### Literally binary

At the core, a binary format is raw `1`s and `0`s. A `1` or `0` is called a bit, and a sequence of them, such as `001001` is called a bit string. Commonly, bits are grouped in words of 8, forming a byte, i.e. `01100001`. A byte can also be called an octet. Sequences of bytes are called byte strings, or octet strings. Bytes can represent different things, such as an integer, a character, or whatever really - it just depends on the encoding.

If you try to look at a raw byte string commonly it looks something like `��d�DU)�` (because you're trying to view raw bytes with utf8-representation). Not very useful. Bytes become useful when they are represented correctly (ie a png picture should be "read" as a png picture, not a utf-8 string). However, some representations are very simple and thus useful in crypto, where you often work with raw bytes that just represent large numbers.

### Hex

A byte can hold one of 256 values (`2^8 = 256`), and can thus be represented as two hexadecimal digits (`16^2 = 256`). However, as we normally represent numbers as 0-9, we can only work with 10 numeric characters. Therefore, the letters A-F are introduced to represent the six missing digits, so hexadecimal numbers might look something like `0A 0F 3C`. It's very common to make byte strings readable in hex. There can be spaces between words (such as the former example) or not (such as `0A0F3C`). The casing doesn't matter in hex, so `0A0F3C == 0a0f3c`. Hex is sometimes prefixed with `0x` to indicate it's indeed a hex string - ie `0x0a` or `0x0a0f3c`, but `0x` is _not_ part of the actual value and `x` is not a valid "digit".

### Base64

Another format is base64, which consists of all upper- and lowercase alphanumerics in ascii, `+` and `/` (`a-zA-Z0-9+/`). For example the hex string `0cccd5ac72166379` can be represented as `DMzVrHIWY3k`.

### Try it

Most unix systems come with some really nifty terminal tools to convert between representations.

```sh
# Generate some random bytes
$ openssl rand 10 > /tmp/random-value

# Look at our bytes in utf-8, not useful
$ cat /tmp/random-value
X��|)���_�%

# Convert our bytes to hex
$ cat /tmp/random-value | xxd -i -p
589ed67c2985e8f25ff7 # length: 20 characters (2 per byte)

# Convert hex to raw bytes
$ echo 589ed67c2985e8f25ff7 | xxd -r -p
X��|)���_�%

# Convert to base64
$ cat /tmp/random-value | base64
WJ7WfCmF6PJf9w==

# Convert from base64
$ echo WJ7WfCmF6PJf9w== | base64 --decode
X��|)���_�%
```

### In short

* *ascii `a` == binary `01100001` == decimal `97` == hexadecimal `61` == base64 `YQ`*

## Symmetric and Asymmetric Cryptography

Modern cryptography is based on both symmetric and asymmetric cryptography. In encryption commonly a combination of both, using a Diffie-Hellman Key Exchange, but that's out of scope for these blog posts. The difference is easy to recognize - in symmetric cryptography you're encrypting using a pre-shared key (such as a password, although not all password-based systems use it for encryption), while asymmetric cryptography is based on public/private key pairs.

### Symmetric Cryptography With a Pre-shared Key

Conceptually symmetric cryptography is simple. You're using a pre-shared key as the input for your encryption algorithm when encrypting. When decrypting, the same pre-shared key is used as input for the decryption algorithm. The upside is that symmetric cryptography is usually _very fast_, and sometimes even has [hardware support](https://en.wikipedia.org/wiki/AES_instruction_set). However, both parties need to know the pre-shared key. Without a secure - already encrypted - channel, there's no way of sharing that key without the risk of a [man-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) intercepting it, leaving us with a catch-22. It is possible to get around this using a Diffie-Hellman Key Exchange, but again - out of scope.

### Asymmetric Cryptography With a Key Pair

With asymmetric cryptography, we have access to a public and private key pair. The public key can be freely shared with other parties, while the private key must remain - well - private to you.

  * **Note** - *The private key is sometimes called Secret Key, so it can be abbreviated "SK" without colliding with the Public Key abbreviation "PK"*


The public key can be derived from the private key, but not the other way around, which is essential for asymmetric crypto to work. This is achieved by using mathematical functions that are easy to perform in one direction but hard to reverse, such as [modulo](https://en.wikipedia.org/wiki/Modulo_operation). However, most asymmetric cryptography algorithms are far more sophisticated than purely using modulo. To understand this concept more in detail, let's make a very simplified example:

To begin with, let's understand modulo by example. In short, modulo gives the remainder of division.

  * **Note** - *The modulo operator is commonly `%` in programming languages, including this example.*

```txt
Here are some modulo examples:

1 % 3 = 1
2 % 3 = 2
3 % 3 = 0
4 % 3 = 1
5 % 3 = 2
6 % 3 = 0
```

These are very low numbers, but given the result of the modulo operation, it is hard to know which were the numbers on the left side that computed the result.

The simplest form of an asymmetric key pair (although very insecure) could be defined as the following:

```txt
q = 17                  // must be a prime to not risk our public key becoming 0
p = 7
SK = q || p = 17 || 7   // || means concat, not or, in this example
PK = q % p = 17 % 7 = 3
```

So our private key is the combination of `q` and `p` (`17` and `7`), and the public key is derived as `q % p`, `17 % 7` - meaning it becomes `3`. Real world algorithms, such as elliptic curves and RSA (Rivest–Shamir–Adleman), are much more sophisticated, and, of course, use much bigger numbers (RSA for example commonly uses 2048 or 4096 bits). But this should give you an idea of how public and private keys are related.

The downside of asymmetric cryptography is that encryption and decryption becomes _very slow_. Commonly algorithms are not even capable of encrypting arbitrarily-sized payloads. This makes it ill-suited to directly encrypt messages with asymmetric crypto.

## RSA vs ECC

There are different types of asymmetric cryptography algorithms, RSA and Elliptic Curves (notice plural, there's multiple types of ECC) being the major players (I'm not even sure there are other options? Probably there are though). RSA is older and uses fairly simple mathematical functions, which I will not dive in. It is easier to crack with enough computing power. That can be countered by using larger key sizes. However, larger key sizes mean more payload to transfer and more work to perform. This is why Elliptic Curves fascinate me. By using more sophisticated algorithms you can achieve the same (or stronger) security with smaller key sizes!

## Elliptic Curves

So, what is an Elliptic Curve anyways? In short, it's a mathematical formula for generating key pairs and performing encryption and signing (and I think more), but I'll leave the mathematical explanation to someone else, e.g. the [Wikipedia article on Elliptic Curve Cryptography](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography). What I will dive into, instead, is how to use them.

I try to think of Elliptic Curve Cryptography as a category of cryptographic algorithms. I will only cover one here, but there's many, many, of them with various support in different systems. ECC is a fairly new concept, so adoption is still pending in some places.

According to [malware.news](https://malware.news/t/everyone-loves-curves-but-which-elliptic-curve-is-the-most-popular/17657) (a source I'm not sure about the trustworthiness of), the most common curves are NIST P-256 and Curve25519. That fact feels aligned with my experience of usage though, so I'll take their word for it. Anyways, for now I will only use NIST P-256 - to not make an already confusing subject more confusing.

### NIST P-256

NIST P-256 is an Elliptic Curve whose keys can be used both for encryption and signing. It was developed by - surprise - NIST (\[US] National Institute of Standards and Technology, thus closely with the NSA). Some [claim it has optimizations that may be for security](https://safecurves.cr.yp.to/). If you're into conspiracy theories, you might think it was designed to have a "backdoor". Idk whether or not that is true, but if you're a conspirator you might want to opt for Secp256k1, which claims to be without the security-degrading optimizations, or Curve25519 which is thought to be less likely to have a backdoor. However, as NIST P-256 is by far the most commonly used curve, and I'm not much of a conspirator, I will go with it.

* **Note** - *The NIST P-256 curve is also known as Prime256v1 and Secp256r1, and not to be confused with Secp256k1 (which is different) - because why have one name when you can have three* 🌞

## Hashing

The concept of hashing is very important in crypto. I just think of hashing algorithms as another set of functions that are easy to perform but hard to reverse.

They do have some other important traits. Specifically:

a) Given the same input, they will always give the same output.
b) The output is always the same number of bytes (ie 32 for SHA256).
c) Making even a small change to the input will change the output drastically.
d) For hashing algorithms considered secure (such as SHA256) you cannot predict or reverse the hash without a lookup table. So it's hard/impossible to derive the input value from the output value.

Some algorithms include MD5, SHA1 and SHA256. MD5 is generally considered broken because it's relatively easy to generate hash collisions (two different inputs that produce the same output). The same goes for SHA1, except for when used in Message Authentication, but that's for another chapter. SHA256 on the other hand is still considered secure.

```txt
Some hex encoded examples of SHA256 checksums
SHA256("a")                   -> ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb
SHA256("abc")                 -> ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
SHA256(my website's png logo) -> 2c9be80039a4a8a8517fe68f23763defbb00ead0ae4da5a803f304a0e617cc0d
```

* **Note** *Take careful note of the input format here. Inputting literally the ascii char `a` will give the checksum  `ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb`. However, `a` can also be represented as `0x61` in hex, but inputting literally `61` will give the checksum `d029fa3a95e174a19934857f535eb9427d967218a36ea014b70ad704bc6c8d1c`. It is thus good practice to use the raw representation of the thing you're hashing whenever possible.*

### Try it

You can use the OpenSSL CLI to generate SHA sums.

```sh
$ printf a | openssl sha256 # sha256("a")
ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb
# We're using printf instead of `echo` because `echo` will include a newline
# which produces a different result:
$ echo a | openssl sha256 # sha256("a\n")
87428fc522803d31065e7bce3cf03fe475096631e5e07bbd7a0fde60c4cf25c7

# as you know, these strings are just different formats,
# so this is also true
$ printf 61 | xxd -r -p | openssl sha256 # sha256(hexToAscii(0x61))
ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb
# ☝️ same result as sha256("a")
```

## Continue

In {% post_link "self-taught-ecc-guide-part-2-hands-on" "part 2" %} I will cover some implementation details and how to apply this knowledge with the help of OpenSSL's CLI.
