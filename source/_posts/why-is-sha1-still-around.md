---
title: Why is SHA-1 still around?
date: 2022-12-08 00:52:04
tags:
- cryptography
- explained
---

If you've been using hashing algorithms in the last few years you (should) have learned to stay away from SHA-1 - because it is considered broken (see eg. [SHAttered](https://shattered.io/)). "Why?" you will ask - and I will answer. It's because it's easy to generate collisions of the hashed results. If you're altering the input in a clever way, you can make SHA-1 give the same checksum, even though the content is different.

In short - stay away from SHA-1.

BUT - as I was browsing through the [X.509 standard (IETF RFC 5280)](https://datatracker.ietf.org/doc/html/rfc5280) (the one powering Public Key Infrastructure certificates; how HTTPS certificates work) the other day I noticed something weird in section [4.2.1.2 (Subject Key Identifier)](https://datatracker.ietf.org/doc/html/rfc5280#section-4-2-1-2). The Subject Key Identifier identifies the key used in this certificate, and is, among other things, used to construct the chain of certificates - i.e. it's used to know which Certificate Authority (CA) signed this certificate. This means it is critical for HTTPS to work.

In this section, which has not been obsoleted by any follow-up standards, you can find the following description

> Two common methods for generating key identifiers from the public key are:
>
> (1) The keyIdentifier is composed of the 160-bit SHA-1 hash of the value of the BIT STRING subjectPublicKey (excluding the tag, length, and number of unused bits).
> ...

This got me very alarmed - if IETF recommends using SHA-1, reasonably that will be adopted very widely. Isn't SHA-1 broken? Does that mean the whole PKI (Public Key Infrastructure) is broken?

Spoiler - PKI/HTTPS is not broken.

As it turns out, to generate a SHA-1 collision you must add something to the altered input. Take the following example:

```
1) SHA1("data") = abc123
2) SHA1("forged data") = 321cba (different than 1)
3) SHA1("forged data" + "clever addition") = abc123 (same as 1)
```

In the case of a key fingerprint (like SKI) the input (public key) is directly derived from the secret key, which is - well - secret. The public key is only valid together with the secret key, and can thus not be altered with any clever additions without it becoming invalid. Thus, it's impossible (or at least very very hard) to generate collisions. As lack of collision-resistance is what broke SHA-1, it is not broken in this case. 

This, of course, also means SHA-1 is still secure for HMAC, as long as the MAC key remains secret. Because you can't generate a hash collision without knowing the full source of data, it becomes impossible to generate collisions.

My advice would still be to stay away from SHA-1 unless you really need to, but if you would ever encounter key fingerprinting or HMAC using SHA-1, you can safely know the application is not at risk because of that :)

Hope you found this as interesting as I did!