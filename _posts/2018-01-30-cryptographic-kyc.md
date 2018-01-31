---
layout: post
title: Proposal for Cryptographic KYC
tags:
- Cryptography
- Banking
---

Quite a lot of KYC could be handled faster & safer if we could prove something _not_ to be true.

Right now KYC involves 

1. aggregating user data in a central database, like [World-Check](https://risk.thomsonreuters.com/en/products/world-check-know-your-customer.html);
2. subscribing to a data provider; and
3. querying the database to see if a potential customer is listed.

Point 1 presents several concerns.

- User data can be compromised in a breach;
- multiple KYC providers expend (read: waste) effort to cover the same demographics;
- businesses wishing to do KYC screening are beholden to the data providers' APIs.

## Bloom Filter

What if we used a bloom filter whose elements are the `full name + birthday` of each individual in an existing database who has some form of demerit?
To store an offender registry the size of the United States, [you'd require < 1GB of space with `p = 0.00001` of a false positive](https://hur.st/bloomfilter?n=323000000&p=0.00001) given 17 hashing functions.

Benefits include that

- in the event of a breach, no user data would be compromised;
- data can therefore be distributed via technology like [IPFS](https://ipfs.io/) or [Holochain](https://github.com/metacurrency/holochain), rather than siloed; and
- KYC providers focus on looking into legitimately suspicious activity (better signal to noise ratio).

When there's a no positive match for a prospective customer, he can be onboarded right away. 

When there is a match, the KYC provider determined to have uploaded 
hashed data to the (hopefully distrubted) file store can be engaged to continue the investigation. Onboarding becomes easier for everyone.

## Boolean Tests

Since the cost of storing data is so low, there's no need to have a vague "wrong-doers" database; we can create different file stores by offense (and still maintain user privacy).
There could be several databases (bloom filters) checking for traits like politically exposed personhood or a history of money laundering.

## Increasing Security & Monetizing

The system described so far falls short in two ways

1. KYC providers have no power to monetize the information they provide. They are only paid when a positive match is detected and their services are required. As well as being unfortunate for them, this creates an incentive to synthetically increase the false positive rate (by adding random 1-bits).
2. While user security is in tact in the sense that a download of the database doesn't allow one to print a list of users suspected of some wrong doing, there's nothing to stop an observer from checking up on an individual user, given her name & birthday.

We can solve both of these problems by modifying the preimage of the hash entered into the bloom filters with a KYC provider's secret key. To upload some user to a bloom filter database, the KYC provider would take `preimage = HMAC(first name || last name || birthday, secret key)` and then enter `preimage` into the bloom filter. To query the data, a business would hit the KYC provider's API with `first name || last name || birthday` (presumably for some cost) and receive the `preimage`. The business would then use the `preimage` to search the database.
