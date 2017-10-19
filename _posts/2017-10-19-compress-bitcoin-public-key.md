---
layout: post
title: Compress a Bitcoin (Elliptic Curve) Public Key
tags:
- Bitcoin
- Crypto
- Software
---

In Bitcoin, a public key is composed of its (x, y) coordinates on the secp256k1 elliptic curve. Since the curve's equation is known (y<sup>2</sup> = x<sup>3</sup> + 7), given only _x_ we can derive _y_ or _-y_. Therefore we can cut the space taken by the public key in half by removing the _y_ value. Since Bitcoin's curve is over a prime field, we can use y's parity to distinguish between the two square root solutions. (If this is confusing, see [this excellent answer](https://bitcoin.stackexchange.com/a/41664).) The compressed public key is then a flag followed by the _x_ value.

The uncompressed serialization format is

```haskell
04 [32 byte x vlaue] [32 byte y value]
```

and the compressed format is

```haskell
[03 if y is odd else 02] [32 byte x value]
```

In Python:

```python
>>> pk = "04a1af804ac108a8a51782198c2d034b28bf90c8803f5a53f76276fa69a4eae77f3010ba699877871e188285d8c36e320eb08311d8aecf27ff8971bc7fde240bfd"
>>> pk_bytes = bytes.fromhex(uncompressed_pk)[1:] # Strip the 04 prefix
>>> x, y = pk_bytes[:32], pk_bytes[32:]
>>> parity_flag = (b'\x03' if int(y.hex(), 16) & 1 else b'\x02')
>>> (parity_flag + x).hex() # The compressed pk
"03a1af804ac108a8a51782198c2d034b28bf90c8803f5a53f76276fa69a4eae77f"
```


