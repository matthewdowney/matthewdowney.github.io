---
layout: post
title: Exchanges, How Do You Feel About an Average Execution Price Limit Order?
tags:
 - Finance
 - Market Microstructure
 - Exchanges
 - Trading
---

A decidedly contrarian friend posed the question, about the protagonists of 
Michael Lewis's 
[_Flash Boys_](https://www.amazon.com/gp/product/0393351599/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0393351599&linkCode=as2&tag=matthewdowney-20&linkId=503ade3ef618d0b075d315d0cc3bf4a5),
"if they were so concerned about the execution price, why didn't they just use 
limit orders?"

He's not alone. Matt Levine voices similar skepticism about 
[IEX](https://en.wikipedia.org/wiki/IEX), whose purported raison d'être is to 
protect the little guy from predatory market makers, in 
["'Flash Boys' Exchange Isn't About the Little Guy"](https://www.bloomberg.com/opinion/articles/2016-02-25/-flash-boys-exchange-isn-t-about-the-little-guy):

>  I try not to give investing advice but I will say: If you are a retail 
   day-trader, consider using limit orders! You can use marketable limit orders 
   -- like, bid up to $41 for that $40 stock if you want -- to make sure that 
   you'll get an execution, but it's a scary world out there, there are a lot 
   of flash crashes, and you don't want to put in a market order at the exact 
   second that the stock spikes to $999.99.
   
I tend to agree with his commentary on IEX, but it did bring to mind a solution 
for an inconvenience with limit orders that we face here at 
[Sixtant](https://sixtant.io).

## Limit Orders Aren't Useful in Thin Books

The problem is that limit orders are in some ways (paradoxically) less useful 
for controlling average execution price in thin books with liquidity dispersed 
across price levels than they are in, say, well-established equities markets. 

This is especially true for cryptocururency order books, where liquidity 
problems are exacerbated by a lack of attention paid to tick sizes, since the 
available liquidity is often not concentrated at the touch. (Though it's a fine 
line — the inverse is true for 
[some markets](https://www.bitmex.com/app/seriesGuide/TRX) 
where the tick size is much too large.)

This type of order book topography is inconvenient for us because we sometimes 
act as market makers in small and non-integrated markets, and hedge our 
positions by trading _against_ other market makers in large, well-integrated 
markets where we need to control our execution price. We execute our taker 
order just as Levine suggests — with a marketable limit order — but still face
risk when setting the limit price.

How should you set the price for that limit order if you know that you're going 
to walk the book? Imagine that you're looking at buying 900 units on the 
following order book:

| Price |  Qty |
|-------|------|
|   102 | 1000 |
|   101 |  500 |
|   100 |  300 |
|    -- |   -- |
|    99 |  300 |
|    98 |  500 |
|    97 | 1000 |

It's going to cost you $100 per unit for the first 300 units, $101 for the next 
500, and $102 for the final 100 units of your order. That's an average price 
of `(100 x 300 + 101 x 500 + 102 x 100) / 900`, or $100.7778 per unit.

But if you set that as your limit price, you'll only execute the first 300, 
rather than the whole 900. To execute the whole 900, you have two options: 
either you

1. enter a marketable limit order to buy 900 @ $102 (over 1% above the average
   price that you want!); or
2. fill the price levels one-by-one, first issuing an order to buy 300  @ $100,
   waiting for it to fill, then issuing another bid for 500 @ $101, etc.
   
Option (2) is really more of an option in the theoretical sense — I'd bet that 
when the first two levels are picked off in quick succession, the probability 
of the rest of the market pulling back outweighs your risk of the average 
execution price changing in option (1). Plus cryptocurrency exchanges are not 
exactly "fast"; each limit order fill could take on the order of 100ms.
   
Option (1) is still obviously better than a market order; you replace infinite 
risk of market impact with a known, finite worst case. Still, you never know, 
maybe the sell order for 500 @ $101 is cancelled right as you submit your order, 
and you end up executing at $101.334. 

## Average Execution Limit Order

So, exchanges, why not offer a limit order variation whose limit is not a "worst
execution price limit" but a "worst average execution price limit"? 

It's a small change, and even plays nicely with existing 
[FOK](https://www.investopedia.com/terms/f/fok.asp), 
[IOC](https://www.investopedia.com/terms/i/immediateorcancel.asp), 
and [AON](https://www.investopedia.com/terms/a/aon.asp)
order types, in addition to the vanilla 
[GTC](https://www.investopedia.com/terms/g/gtc.asp)
limit order, though it does imply some extra CPU cycles in the matching engine 
spent checking the average order price as it walks the book.
