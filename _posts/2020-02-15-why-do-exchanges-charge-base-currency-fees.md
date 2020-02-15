---
layout: post
title: Why do Exchanges Charge Base Currency Fees?
tags:
 - Trading
 - Markets
---

Some [prominent](https://www.binance.com/) cryptocurrency 
[exchanges](https://bitso.com/) charge fees in each market's base currency, 
rather than the counter (quote) currency, and I've never quite gotten a 
satisfactory answer as to why. More confusingly, they only do so on one side of 
the book (when buying).

My problem with this is twofold: one, there's a slightly different transaction
cost for buying, which is not significant in magnitude but is conceptually 
perturbing; and two, why not charge fees denominated in the asset that you're 
_already_ using as both a 
[unit of account and a medium of exchange](https://www.imf.org/external/pubs/ft/fandd/2012/09/basics.htm)
on the market itself?

> E.g. the price to buy `qty` units of "ABC" on an ABC/USD market is 
  `(qty / (1 - fee)) * price` in this scheme, compared to 
  `qty * price * (1 + fee)` in a "normal" fee structure.
  

I asked a friend formerly employed by one such exchange, and he framed the 
decision in terms of common sense ("shouldn't the fee be taken out of the thing
that the user receives?") and user experience ("if you have _just_ enough money
to buy some quantity, wouldn't it be confusing if you needed slightly more to 
account for the fee?"). 

As a writer of trading algorithms, I find it a significantly _worse_ user 
experience than one using a normal fee structure, but I see where he's 
coming from! Plus I guess I do appreciate it in terms of like, gamesmanship.
It's _slightly_ harder to develop abstractions to insulate pricing logic from 
the details of a more complex fee structure — which affects both the price _and_ 
quantity, so they have to be considered together — and therefore those who do so 
might have a slight advantage compared to those who overlook it. (Okay, this is 
probably insignificant, but part of being an engineer is, after all, the 
insatiable desire to solve utterly useless problems that are just difficult 
enough to be fun.)

Still, I want to know _why_. Somebody is in charge of thinking about markets 
(and fees, the main revenue source!) at these exchanges.

I've heard tax evasion (that notorious 
[impetus of human ingenuity](https://www.researchgate.net/publication/329773795_Jeanne_Calment_the_secret_of_longevity))
offered as one speculative explanation (tax treatment of digital things that are 
not strictly money is still not totally clear), but it doesn't feel realistic 
that cryptocurrency exchanges could just not declare _half_ of their fee revenue.

It also occurs to me that this practice could be ideologically motivated. Maybe 
there's correlation between being the type of person that starts a cryptocurrency
exchange and being the type of person who thinks cryptocurrencies are sounder 
monetary instruments than fiat currencies — probably exhibiting a higher r^2
than a correlation with being fascinated by market structure as an algorithm 
for human cooperation — in which case using cryptocurrencies to denominate fee 
payments would make perfect sense. In fact that would be an excellent 
explanation, if and only if, they charged base currency fees on _both_ sides of
the trade!

I've also heard it framed as a speculative bet. Maybe these exchanges want 
to build up exposure to each of their listed assets, but also be sure to collect 
some revenue in the same money they use to pay rent and salaries. If this were
the case, wouldn't it make more sense for them to employ a normal fee structure,
and then systematically exchange some percentage of their collected fees on the
open market? It would even boost their volume a little bit.

The most sinister explanation I can think of is that it could be a way to take 
advantage of the adverse selection faced by those offering liquidity, 
systematically and transparently front running their most informed users. The 
best way to front run a gargantuan buy order is of course to have _already_ 
bought at the very same price as that order, not to trade later against the 
post-execution order book. This is the only situation I can think of where it 
_does_ makes sense that the fee currency would change with the side of the order.

But this also seems wildly unlikely — why would exchanges look for that type of 
edge, which might be probabilistically significant but would nevertheless be 
negligible given the quantities at which they could execute? And, more 
fundamentally, the correct way to do this would be to vary the fee currency with
the aggression of the order in addition to its side. To take maximum advantage 
of the maker's adverse selection risk, you'd want to always charge both the 
maker and the taker fee denominated in the asset that the taker is receiving. 

There's no other market I can think of where this happens — commodities markets, 
even those that settle in the underlying, don't take in-kind commissions, and 
for obvious reasons bookies, art galleries, and auction houses don't operate in
that manner either. If anybody out there can shed light on this mystery, please
email me!
