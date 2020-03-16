---
layout: post
title: Effects of Derivative Market Structure on Global Liquidity
tags:
- Trading
- Market structure
---

Three days ago, Bitcoin joined global equities markets in a free-fall. Perhaps 
even more intriguingly, we saw abnormal distortions in derivatives markets 
before and during the crash.

I want to offer some observations about data that we collected here at 
[Sixtant](https://sixtant.io/).


## 04:30 — 06:00 CST: Market Distortion

During the early morning hours of Thursday, March 12th, the futures contracts 
listed on BitMEX entered backwardation during a BTC price crash from ~7.5k to 6k 
on the perpetual swap market.

![early morning futures backwardation](/static/img/backwardation.png)
<center><i>XRP futures for March 27th were over 6% cheaper than spot XRP for the 
better part of an hour.</i></center><br>

Backwardation (or contango, its inverse) to this extent is not  “normal” market 
behavior — arbitrageurs should step in to close the price gap between the 
futures and the spot. 

Usually the balance of order flow here is between traders who want to go short 
(or long) with leverage during a big price movement and are willing to pay a 
premium to do so, and arbitrageurs who decide if the liquidity cost (or cost of 
carry) until the future’s expiry is worth the premium. During price drops these 
traders tend to go short, and in price jumps they go long. 

One can only speculate as to why this market distortion persisted in the first 
place, but a possible answer is that there are less traders prepared to 
arbitrage a future in backwardation, since the trader needs to already own the 
spot asset. 

In contango, on the other hand, the arbitrageur only needs the settlement 
currency of the future, since they cover by buying the asset on the spot market, 
rather than selling.

So when speculative market participants go long, arbitrageurs step in to keep 
prices in equilibrium, but when the inverse happens in such a precipitous crash, 
there are fewer participants who can provide liquidity.

## Effects on Order Book Liquidity

This is more of a hypothesis than a proven effect, but there is a narrative 
here that presents an explanation for how this price crash and backwardation 
may have made the global market, not just BitMEX, more fragile and set the stage
for the rest of the day.

![changing bid-ask spreads during the crash](/static/img/bid-ask-spreads.png)

The first component is that such backwardation may have caused market makers 
to reduce their position sizes, removing a significant amount of crypto 
liquidity from the global market early in the day. 

Traders (speculators as well as market makers) often create synthetic positions 
to allow them to trade assets they don’t own without the price risk. For 
example, you might have an inventory of 1 BTC and a desire to trade XRP, in 
which case you could use 0.5 BTC to short XRP futures and the other 0.5 BTC to 
buy XRP on a spot market. 

Market participants had most likely created their positions by shorting futures 
at a premium of 1-3%, but thanks to the backwardation in the morning they were 
now seeing an opportunity to close their short positions several percentage 
points below the spot price. 

Such extreme backwardation doesn’t usually last, so traders probably rushed to 
close or reduce their positions early in the day at a healthy profit, with the 
expectation of reopening them when the futures reached par with spot markets. 
However, the futures basis stayed mostly negative throughout the day. 

The second liquidity reducing dynamic is that, in that same way that some 
traders were incentivized to take profit on their short futures positions, 
anybody who was long began to worry about liquidation. 

These might have even been the same traders, since in many cases (certainly on 
BitMEX), unrealized P&L is not counted toward the cross margin collateralizing 
other open positions.

It's possible that many of the open long positions — at varying levels of 
leverage — for the USD settled BTC futures were used to fabricate dollars in the 
same way that short positions are used to create crypto. Prudent traders seeking
to mitigate their risk of liquidation would have reduced or closed their open 
positions, thereby removing dollars from the order book as well.


## 20:00 — 00:00 CST: Carnage

Later in the day, liquidations on BitMEX sent the spot price into a free-fall 
and pushed the BTC futures more than 15% into backwardation. 

![liquidations later in the day](/static/img/liquidations.png)
<center><i>5-minute moving averages of different futures contract basis — 
individual ticks were even more extreme.</i></center><br>

![bid-ask spreads on bitmex](/static/img/bitmex-bid-asks.png)
<center><i>Though BitMEX's average bid-ask spread is much lower than Bitstamp's, 
the extremes reached during the liquidations were impressive. Check out the 
"max" value for the H20 futures, it's 13.4%, and that's for a 10 minute moving
average!</i></center><br>

## Market Fragility

The standard explanation for events like this goes something along the lines of:
liquidations on platforms like BitMEX are self-reinforcing. Long positions are 
liquidated via market dumps, which not only puts downward pressure on _that_ 
order book, but on the global market as well. Since price discovery happens on 
BitMEX, arbitrageurs buy there and then dump on other spot markets, sinking the 
price globally. Since crypto markets exhibit nominal rigidity (price 
"stickiness"), the subsequent drop in price on spot markets is not followed by 
a correction, but instead serves only to affect the indices used for marking the 
swap and futures products, perpetuating a cycle of liquidations.

This could certainly be true, and without mutual accountability between 
counterparties, a margin call system isn't viable, so it's hard to imagine 
what structural changes could be made to improve the situation.

At the same time, I'm fascinated by the extreme futures backwardation that 
preempted the evening's price crash. Maybe it's just 
[good old human apophenia](https://en.wikipedia.org/wiki/Apophenia),
but I have to think that there's reason that the headline here was "Bitcoin 
Price Crash" and not "BitMEX Flash Crash".

Who knows though, maybe some gargantuan fund out there was selling BTC a little 
too exuberantly to make margin for leveraged positions in equities, and the 
lesson is to not even try to explain the emergent properties of complex systems.
