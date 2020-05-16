---
layout: post
title: Encrypting Keys in Clojure Applications
tags:
 - Software
 - Clojure
---

I think the first time that I had to deal with API keys as opposed to tokens — 
i.e. where the secret wouldn't 
[just be given away](../security-flaws-exchange-apis-consider-during-design.html) 
inside of the outgoing API call anyway — was when I started contributing to an 
open source abstraction layer ([XChange](https://github.com/knowm/XChange)) for 
different cryptocurrency APIs in 2015.

At the time, the advice that I got was:

```java 
private final String key = System.getenv("API_KEY");
private final String secret = System.getenv("API_SECRET");
```

It wasn't until those API keys started having access to more money that I began 
to rethink the convenience of environment variables.

## Workflow With ~150 Lines of Clojure

Instead of environment variables -- which are accessible from other processes 
(that's the point, right?) and could feasibly end up in a debug log -- I've 
adopted the following workflow:

1. Generate a new set of API keys.
2. Read my encrypted map of keys from disk, decrypt it with a passphrase, 
   `assoc` in the new key & secret, encrypt it again, and write it to disk.
3. At the entry point for my application, use `(.readPassword (System/console))`
   to securely read in the passphrase, and then use it to decrypt the key file
   and read it into a Clojure map. 
4. Instead of passing the key map around (allowing it to potentially escape 
   into a debug log, or be printed at the REPL if I do something dumb), the top 
   level code of my application passes the credentials into a `signer-factory` 
   for each api that closes over the credentials.
   
   ```clojure
   ;; The factory is shaped something like this
   (defn request-signer-factory 
     [{:keys [key secret]]
     (fn [request-to-sign]
       (sign-request request-to-sign key secret)))
   
   ;; Then an API endpoint looks like this
   (defn place-order! 
     [signer {:keys [price qty side market post-only?]}]
     (let [request (comment "Format the order data for the exchange")
           signed (singer request)]
       (do-http-request! signed)))
   ```
   
I like this workflow more than others which are centered around only encrypting 
credentials inside of your Git repository, and decrypting them when you clone / 
pull, because it means that not even on my development machine are keys just 
sitting around in plaintext.

> To skip straight to the implementation source, head over to [this gist](https://gist.github.com/matthewdowney/d5d816a0274ea2d1fd5e9eab4a933e57).
   
## Conveniently Editing the Encrypted Keyfile

The reason that I can call (2) "convenient" with a straight face is that it's
easy to have an interface that's similar to how `git commit` with no `-m` flag
works.

It gets the password, pops up the system's default text editor (vim for me), 
and then encrypts & writes the editor contents once you close it:

```clojure
(defn edit-keys!
  "Run from a terminal (maybe via a lein alias) to edit the 
  encrypted API keys with the system's default text editor."
  []
  ;; Implementation for each of these in the next section
  (-> (read-keys)
      (update-password?)
      (edit-keys)
      (write-keys))
  (System/exit 0))
```

<br>

![editing some API keys from the terminal](/static/img/key-edit.gif)


## Base Implementation

I'm going to use [funcool/buddy-core](https://github.com/funcool/buddy-core) 
for the cryptography. On top of buddy's API, we can build key stretching, 
encryption, and decryption in just a few lines of code:

```clojure
(require '[buddy.core.codecs :as codecs]
         '[buddy.core.nonce :as nonce]
         '[buddy.core.crypto :as crypto]
         '[buddy.core.kdf :as kdf])
(import '(java.util Base64))

(defn bytes->b64 [^bytes b] (String. (.encode (Base64/getEncoder) b)))
(defn b64->bytes [^String s] (.decode (Base64/getDecoder) (.getBytes s)))
```

#### Key Stretching

```clojure
;; Take a weak text passphrase and make it brute force resistant
(defn slow-key-stretch-with-pbkdf2 [weak-text-key n-bytes]
  (kdf/get-bytes
    (kdf/engine 
    {:key weak-text-key
      ;; Keep this constant across runs
     :salt (b64->bytes "j3gT0zoPJos=")
     :alg :pbkdf2
     :digest :sha512
     ;; Target O(100ms) on commodity hardware
     :iterations 1e5})
    n-bytes))
```

#### Encryption

```clojure
(defn encrypt
  "Encrypt and return a {:data <b64>, :iv <b64>} that can be 
  decrypted with the same `password`."
  [clear-text password]
  (let [initialization-vector (nonce/random-bytes 16)]
    {:data (bytes->b64
             (crypto/encrypt
               (codecs/to-bytes clear-text)
               (slow-key-stretch-with-pbkdf2 password 64)
               initialization-vector
               {:algorithm :aes256-cbc-hmac-sha512}))
     :iv (bytes->b64 initialization-vector)}))
```

#### Decryption

```clojure
(defn decrypt
  "Decrypt and return the clear text for some output of `encrypt` 
  given the same `password` used during encryption."
  [{:keys [data iv]} password]
  (codecs/bytes->str
    (crypto/decrypt
      (b64->bytes data)
      (slow-key-stretch-with-pbkdf2 password 64)
      (b64->bytes iv)
      {:algorithm :aes256-cbc-hmac-sha512})))

(-> (encrypt "some clear text" "my password")
    (decrypt "my password"))
;=> "some clear text"
```

## Full Implementation

Have a look at this [gist](https://gist.github.com/matthewdowney/d5d816a0274ea2d1fd5e9eab4a933e57) for a step by step on how you might use this
in a project, or just read the clj source here to dive straight in:

<script src="https://gist.github.com/matthewdowney/d5d816a0274ea2d1fd5e9eab4a933e57.js?file=client-encrypt.clj"></script>

## Clearing Keys From Memory

How far do we want to take it? Should we clear the API keys from memory?

In my case, this doesn't make sense — my request signer needs to hold on 
to the keys and sign requests throughout the lifetime of the application. 

If, on the other hand, you were working with something where you just needed
to use the keys once, at startup, you could minimize the time that keys spend
in memory by (1) making sure they're never `String`s, only `char[]`s, and (2) 
writing over the `char[]`s as soon as you're done with them. 

> See: [explanation](https://www.sjoerdlangkemper.nl/2016/05/22/should-passwords-be-cleared-from-memory/) 
  of how this works on the JVM.

Security is a spectrum — you can decide how far you want to go. But at the very
least, prefer client-side encryption to environment variables — it's easy, 
especially in Clojure.

