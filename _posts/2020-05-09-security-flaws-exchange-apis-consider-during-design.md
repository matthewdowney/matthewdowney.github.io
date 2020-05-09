---
layout: post
title: Security Flaws in Exchange APIs — Things to Consider in the Design Phase
tags:
 - Software
 - Crypto
---

Part of an exchange's job is providing programmatic access to its trading 
platform, which requires that its API include a mechanism for authenticated 
calls.

> With the notable exception of one Russian "VC" backed exchange founder who 
  legitimately told me that they would offer no API "because otherwise the bots 
  will quickly take us down".
  
This mechanism should, ideally, be secure. I've integrated trading or data 
collection systems with at least 50 exchange APIs over the last 5 years, so 
I've seen some things. My hope — slim though it may be — is that somebody on 
the dev team for the next upstart exchange will consider the following issues 
before designing its API.

Since at [Sixtant](https://sixtant.io/) we make markets using these APIs, I'll 
also give a market maker's perspective on each practice.

## Use a Secret Key, Not a Token

Okay, this one is admittedly basic, but I've seen it disregarded. It's not 
really 
[a key in the cryptographic sense](https://en.wikipedia.org/wiki/Key_(cryptography\))
if the authentication mechanism boils down to "send this string that both of us 
knows along with every request"; it's a token. Tokens are bad, signatures are 
good, with or without HTTPS.

Each API request needs to make use of two keys: a public key which is included 
in the request, and a secret key which is used to sign the request.

As a market maker, why would I use an API where any request logging on the 
exchange's end would give a casual viewer complete access to my funds? 
Furthermore, such an API imposes unnecessary constraints on _my_ behavior. 
E.g. the component of my system sending requests to the API has to be just as 
secure as the one generating the request. What if I want them on separate 
servers? What extra steps do I have to take to make sure the request forwarding 
server and its logging are secure?

## Use an HMAC, Not Hash(secret | data)

Colin Percival, the author of this list of
[Cryptographic Right Answers](https://www.daemonology.net/blog/2009-06-11-cryptographic-right-answers.html)
which everyone should read, said it in 2009, but it's relevant today:

> Do not design your own way of generating symmetric signatures (e.g., for API 
  requests); especially avoid the common "concatenate key and data, then input 
  to a hash function" approach.

This leaves requests open to a 
[length extension attack](https://en.wikipedia.org/wiki/Length_extension_attack)
which may not be a problem initially given the message format, but is a sneaky
thing to have to think about every time you add a new endpoint.

## Use a Nonce (a Client Order ID Is Ideal!)

Okay, assuming that the request data is properly signed, that data should 
include some form of 
[cryptographic nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce) to 
protect against [replay attacks](https://en.wikipedia.org/wiki/Replay_attack). 
This is tricky to do in a way that's convenient for everybody.

### Increasing Numeric Nonces

For the exchange, it's convenient to make the nonce a monotonically increasing
number. It requires a storage space of 1. Just store the most recently used 
nonce and compare it with the incoming request, and if it's valid, swap them
out. Simple!

For the API users, it's not so convenient, because it means that no request 
can be made before the previous request has completed. One might think that 
as long as the requests are sent out in the right order, with the correct 
nonce values, that they will also inevitably arrive at the exchange / be 
processed in that order — and it's not always true.

> See: [doe-eyed me in 2016](https://github.com/knowm/XChange/issues/250#issuecomment-207109354) 
  encountering this issue in an open source API integration project that I 
  contributed to, laboring under the mistaken impression that if I just got the 
  library to deterministically send the requests in the right order that 
  everything would work. Hah!
  
### Client Order IDs

The ideal solution to this is to use a client order id as a nonce for the order 
placement endpoint, since order execution is precisely the kind of request that
should not be limited in this manner. (Use rate limits!) This means that the 
API user supplies a unique id for their order, which is distinct from the id 
that the exchange generates. Other endpoints can use a monotonically increasing 
numerical nonce scheme.

This has the _enormous_ added benefit that the API user is able to recognize 
their order when it is broadcast over the authenticated WebSocket feed (which 
your API ideally has), even before receiving the response to the placement 
request (where the public id of the order is first communicated). This helps 
avoid an entire class of errors for the API user.

### Time Windows

This is a sledge-hammer solution that I don't feel entirely good about, unless 
it's combined with allowing unique client order ids for order placement 
requests.

It eschews the idea of a nonce entirely, and instead requires that each payload 
specify an expiration timestamp. This is good because the request can't be 
replayed after that time, but it's bad because it can be replayed before that
time. 

If somebody can intercept your request for your open orders, say, it's 
certainly bad. But, if they can intercept the request, they can probably 
intercept the response, too. So, the malicious actor will see your open orders.
However, if they can replay that same request forever, they can _always_ see 
your open orders, and trade using that information advantage. Much worse!

So, in that situation, having an expiry time for the request is good enough.
However, when the request is an order placement without a client order id, an 
expiry time doesn't prevent a disastrous outcome. If an API user is placing
a marketable order to sell $10,000 of something, for example, and the request
expires after 5 seconds, an attacker could force them to sell $10,000 * the 
amount of times they can rebroadcast the request during that time frame. Not 
ideal.

## TL;DR

These are solved problems — just imitate an exchange with an exceptionally well 
put-together API. Here are links to the documentation for two of the best:

- [Coinbase Pro](https://docs.pro.coinbase.com/#introduction)
- [Gemini](https://docs.gemini.com/)