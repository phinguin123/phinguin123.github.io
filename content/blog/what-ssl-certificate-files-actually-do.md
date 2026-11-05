---
title: "What the Four SSL Files Actually Do"
date: 2026-03-26T12:12:00+09:00
draft: false
categories: ["Security"]
tags: ["tls", "ssl", "pki", "nginx", "openssl", "linux", "gemini"]
summary: "cert.crt, chain_cert.crt, root_cert.crt and a .key. I'd installed them a dozen times without knowing why there were four, so I finally sat down and worked it out on my own server."
---

For a long time my whole relationship with SSL was: buy a cert, paste whatever Google or an AI said into the nginx config, reload until the padlock shows up. One day in March I'd had enough:

> I have been just doing whatever google or gemini says but now it is time to fully understand. There are 4 things. cert.crt chain_cert.crt root_cert.crt and .key. How do they work together, what are they exactly, how do they really certify?

Short answer: the `.key` never leaves the server, `cert.crt` and `chain_cert.crt` are what the server sends, and the root file doesn't belong on the web server at all.

## The four files

| File | What it is | Where it goes |
|---|---|---|
| `.key` | My private key. Generated on my server, mathematically paired with a public key. | Server only. Never sent anywhere, not to the CA and not to browsers. |
| `cert.crt` | My leaf certificate: my domain + my public key + the CA's signature over both. | Server; sent to every client. |
| `chain_cert.crt` | The intermediate certificate that signed my leaf. | Server; sent with the leaf. |
| `root_cert.crt` | The CA's root, which signed the intermediate. | Not the web server. Clients already have it. |

When a browser connects, my server sends the leaf + intermediate. Browser checks the intermediate's signature against a root it already trusts, then the leaf's signature against the intermediate, then makes my server prove it has the private key matching the leaf. Any link fails → red warning.

In nginx that means stacking leaf-then-intermediate into one file and pointing at the key:

```bash
cat cert.crt chain_cert.crt > fullchain.crt   # order matters: leaf first
```

```nginx
ssl_certificate     /etc/nginx/ssl/fullchain.crt;
ssl_certificate_key /etc/nginx/ssl/private.key;
```

## Why three certs instead of one

My actual question was more blunt:

> why do I need root, chain and the cert. Can't I have one single authority that will prove that my domain is safe and I just have my cert?

Because of what happens if that one authority's key leaks.

To sign millions of certs automatically, a CA's signing key has to be on an internet-connected machine. If that key were the root — the thing every browser on earth trusts — one breach would let an attacker make a valid cert for any bank or email provider, and the only fix would be distrusting the root, which breaks every site it ever signed.

So the root key stays offline. It's used rarely, to sign a handful of intermediates, and the intermediates do the daily signing. If one gets compromised, the CA revokes that intermediate and the root (and everything else) survives. The chain is basically a way to keep the most valuable key off the network.

That also explained why my machine trusts ~150 roots and not one. There's no single authority of the internet. CAs are competing companies, each with their own roots (often several each, e.g. separate RSA and ECC roots), and a browser needs all of them to visit everyone's sites.

## What "signing" actually means

1. The CA takes the cert's contents (domain, my public key, validity dates…) and hashes them. The cert says which algorithm, e.g. `sha256WithRSAEncryption`.
2. The CA makes a signature over that hash with its private key and attaches it.
3. A client recomputes the hash itself and verifies the signature using the CA's public key (from the intermediate or root it has).
4. If they match, the CA really issued this exact cert and not one byte changed. Swap in a different public key and the hash changes, verification fails.

So yes, the client and CA use the same hash function, and the cert itself tells the client which one.

## Two answers that didn't hold up

### "The root is so IoT devices trust your server"

At one point the explanation for why the CA gave me a root file was that devices like IoT boards "need to trust your server." That felt off:

> The root certificate is to trust the CA and not my server. Which in turn lets the client trust the chain which in turn lets the client trust the cert.crt isn't it? If you are making my server itself the root cert then... no need to chain and cert.crt?

Reply was "You are 100% right… I misspoke." The corrected version is the useful one: the downloaded root is for a client whose trust store doesn't have it, like a minimal device or custom environment. You install it into *that client's* trust store. It never goes on the server, and sending it from the server does nothing anyway — a client can't be made to trust a root just because the server it's checking handed it over. If that worked any attacker could just ship their own root.

### "A bare-bones Linux server has no trust store"

Next claim was that a minimal Linux box is a "blank slate" with no roots. Except:

> Even though I don't have the root cert in my bare-bones linux server, I am still able to curl and communicate with the website.

It isn't blank. Basically every distro ships `ca-certificates`, Mozilla's root list, at `/etc/ssl/certs/`. A minimal image without that package fails with `curl: (60) unable to get local issuer certificate`, but a normal Ubuntu box doesn't.

## Finding my root by hand

I bought from Sectigo, but the only Sectigo-named roots in `/etc/ssl/certs` were an "R" and an "E" one (RSA and ECC), and when I compared them with the `root_cert.crt` from my download, they didn't match. Gemini's attempt at explaining this one is the weirdest part of the whole conversation — the shared chat literally contains its internal reasoning ("Wait, I should not output these thoughts") and then `Response stopped`. Never got an answer lol.

So I looked myself. A cert file is base64 text, and its first characters encode the cert's length and structure, which is distinctive enough to search for (an exact comparison would use a fingerprint, below). Took the start of my downloaded root and searched the trust store:

```bash
grep -Rl "MIIF3jCCA8agAw" /etc/ssl
```

```text
/etc/ssl/certs/USERTrust_RSA_Certification_Authority.pem
/etc/ssl/certs/ca-certificates.crt
/etc/ssl/certs/fc5a8f99.0
```

So my root *is* in the trust store, as USERTrust RSA Certification Authority. I'd just been comparing against the wrong ones. The mystery was boring — my chain goes up to USERTrust, not the Sectigo-named roots.

Why is a Sectigo chain rooted in "USERTrust"? Sectigo is the old Comodo CA, rebranded after an acquisition, and USERTrust is a root it owns. A root's name is part of what's signed and is hardcoded into billions of devices, so it can't just be renamed. The brand on my receipt and the name in the trust store are just different.

And why three hits for one cert? Same root, three ways:

- `USERTrust_RSA_Certification_Authority.pem` is the root as a single file (on Ubuntu it's itself a symlink into `/usr/share/ca-certificates/`).
- `ca-certificates.crt` is every trusted root concatenated into one bundle, for software that wants one file.
- `fc5a8f99.0` is a symlink named after a hash of the cert's subject, so OpenSSL can look up a CA by name directly instead of scanning the folder.

Nothing gets generated per visit. That root was made in 2010 and is valid until 2038.

## Tools that replaced guessing

Grep worked, but these are the proper ones:

```bash
# Who is this cert, and who signed it? (Subject == Issuer means self-signed root.)
openssl x509 -in root_cert.crt -noout -subject -issuer -dates

# Compare two certs exactly, regardless of PEM/DER formatting
openssl x509 -in a.crt -noout -fingerprint -sha256

# Does my private key actually match my certificate? (RSA)
openssl x509 -noout -modulus -in cert.crt     | openssl sha256
openssl rsa  -noout -modulus -in private.key  | openssl sha256

# Does the chain verify, step by step?
openssl verify -CAfile root_cert.crt -untrusted chain_cert.crt cert.crt

# What does my live server actually send?
openssl s_client -connect example.com:443 -servername example.com -showcerts

# Don't reload nginx with a typo in it
sudo nginx -t
```

`s_client` is the most useful one. It shows the chain your server really serves, which is where "works in Chrome, fails in curl" problems (missing intermediate) show up.

## One more correction

Re-reading the TLS handshake explanation for this post, one part is out of date. It described the browser generating a session key, encrypting it with the server's public key from the cert, and the server decrypting it with the private key.

That's RSA key transport, and TLS 1.3 removed it. On TLS 1.2 modern browsers use ECDHE instead. So for a modern browser hitting my site, the session key is never sent at all — both sides run an ephemeral (EC)DHE exchange to derive it, and the cert's private key only signs the handshake to prove the server is who the cert says. The practical difference is forward secrecy: if my private key leaked later, it couldn't decrypt traffic recorded earlier. Under the old model it could.

The big picture Gemini gave was right (identity from the chain, then a fast symmetric key for the traffic). The mechanism was just a decade old.

It also pointed at something to fix: my cipher list was `HIGH:!aNULL:!MD5`, which still allows RSA key exchange for any TLS 1.2 client that asks. Limiting TLS 1.2 to ECDHE suites closes that.

I still buy certs and paste them into nginx. Difference is now I know which file goes where and why, and when something breaks I reach for `openssl s_client` instead of another config snippet.
