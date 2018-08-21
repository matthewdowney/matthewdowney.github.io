---
layout: post
title: Cryptographic KYC
tags:
- Cryptography
- Banking
- Essays
---
Basic cryptography is more than capable of improving today's KYC processes.

In fact, KYC could be handled both faster and safer if we could prove something _not_ to be true (see [Type I/II Errors](https://en.wikipedia.org/wiki/Type_I_and_type_II_errors)). Today, KYC involves aggregating user data in a central database, like [World-Check](https://risk.thomsonreuters.com/en/products/world-check-know-your-customer.html); subscribing various businesses to such a data provider; and querying the database to see if a potential customer is listed.

The centralization of such a database presents several concerns. User data is vulnerable to compromise in a breach (see [Equifax](https://www.ftc.gov/equifax-data-breach) on centralized data storage), multiple KYC providers expend (read: waste) effort to cover the same demographics, and  businesses wishing to do KYC screening are beholden to the data providers' APIs.

What if we used a bloom filter whose elements are the `full name || birthday || SSN` of each individual in an existing database who has a Government Approved Demerit™?

To store an offender registry the size of the United States, [you'd require < 1GB of space with `p = 0.00001` of a false positive](https://hur.st/bloomfilter?n=323000000&p=0.00001) given 17 hashing functions. Benefits include that

- in the event of a breach, no user data would be compromised;
- data can therefore be distributed via technology like [IPFS](https://ipfs.io/) or [Holochain](https://github.com/metacurrency/holochain), rather than siloed; and
- KYC providers are incentivized to focus on cataloging legitimately suspicious activity (better signal to noise ratio).

When there's a no positive match for a prospective customer, she can be onboarded right away. *There is no possibility of a false negative.*

When there is a match, the KYC provider determined to have uploaded hashed data to the (distrubted) file store is engaged to continue the investigation. This is presumably better for KYC providers because they are engaged only in labor intensive—i.e. billable—cases.

Since the cost of storing data is so low, there's no need to have a vague "wrong-doers" database; we can create different file stores by offense (and still maintain user privacy). There could be several databases checking for traits like politically exposed personhood, record of money laundering, or civil suits.

The system described so far falls short in that it creates a perverse incentive for KYC providers and doesn't guarantee user privacy. Both can be solved with a single modification.

The perverse incentive for KYC providers is simple: they have no power to monetize negative matches and might thus decide to inject false positives into the filter (by adding random 1-bits). At the same time, user privacy is gone once a snooper knows their name, birthday, and ssn. 

Both issues are mitigated by modifying the preimage of the hash entered into the bloom filters with a KYC provider's secret key. To upload some user to a bloom filter database, the KYC provider would take 

```haskell
preimage = HMAC(first name || last name || birthday, secret key)
``` 

and then enter `preimage` into the bloom filter. To query the data, a business would hit the KYC provider's API with `first name || last name || birthday` (presumably for some small cost) and receive the `preimage`. The business would then use the `preimage` to search the database. It's obvious how this protects user privacy.

Should the KYC provider wish to perform a false positive injection, client businesses would discover the fraud by comparing the algorithmic false positive probability with their experienced rate of false positives.
