---
layout: post
title: How to Generate a SegWit Bitcoin Address (P2WPKH-P2SH) with Python
tags:
- Bitcoin
- Crypto
- Software
---

[BIP141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) (SegWit) briefly describes the generation of SegWit addresses that are backwards compatible by nesting the pay to witness public key hash (P2WPKH) transaction in a pay to script hash (P2SH) transaction.

> The following example is the same P2WPKH, but nested in a BIP16 P2SH output.
>
> ```haskell
> witness:      <signature> <pubkey>
> scriptSig:    <0 <20-byte-key-hash>>
>               (0x160014{20-byte-key-hash})
> scriptPubKey: HASH160 <20-byte-script-hash> EQUAL
>               (0xA914{20-byte-script-hash}87)
```

The address which can spend such a transaction is then indistinguishable from a normal pay to script hash address (such as a standard multisig address).

Practically, this means that to generate an address, we need just need to prefix the `HASH160` of the described `scriptSig` with the standard p2sh version byte (`0x05`/`0xc4` for main/testnet respectively) and Base58 encode it.


First, translating the standard `HASH160(x) = RIPEMD160(SHA256(x))` to Python:

```python
import hashlib
def hash160(x): # Both accepts & returns bytes
    return hashlib.new('ripemd160', hashlib.sha256(x).digest()).digest()
```

Then we can define a function that accepts a **compressed** public key and generates the corresponding address. For simplicity I'll use the Base58 encoding provided in [bip32utils](https://github.com/prusnak/bip32utils).

```bash
$ pip install git+https://github.com/prusnak/bip32utils
```

```python
from bip32utils import Base58

def p2wpkh_in_p2sh_addr(pk, testnet=False):
    """
    Compressed public key (hex string) -> p2wpkh nested in p2sh address. 'SegWit address.'
    """
    # Script sig is just PUSH(20){hash160(cpk)}
    push_20 = bytes.fromhex("0014")
    script_sig = push_20 + hash160_bytes(bytes.fromhex(pk))

    # Address is then prefix + hash160(script_sig)
    prefix = b"\xc4" if testnet else b"\x05"
    address = Base58.check_encode(prefix + hash160(script_sig))
    return address
```

Then you can go ahead and test it

```python
>>> pub = "03a1af804ac108a8a51782198c2d034b28bf90c8803f5a53f76276fa69a4eae77f"
>>> p2wpkh_in_p2sh_addr(pub)
"36NvZTcMsMowbt78wPzJaHHWaNiyR73Y4g"
>>> p2wpkh_in_p2sh_addr(pub, testnet=True)
"2Mww8dCYPUpKHofjgcXcBCEGmniw9CoaiD2"
```
