---
layout: post
title: Extract XPUB from Ledger Nano S (Bitcoin/Ethereum/any HD crypto)
tags:
- Bitcoin
- Ethereum
- Crypto
- Misc
---

# Define: xpub
An extended public key (xpub) sits within the hierarchy of [BIP32 HD wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki). It can generate any number of public keys & wallet addresses from only the public portion of the xpub/xpriv keypair—ideal for situations where a server must be able to generate deposit addresses, but the compromise of that server shouldn't result in a loss of funds.

![HD wallet diagram](/static/img/bip32.png)

Most wallet software uses this kind of key derivation to keep track of your addresses (usually organized according to [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)).

# Ledger Nano S
While most wallet software makes it simple to export your xpub, so far the Ledger doesn't provide this option in either their Bitcoin or Ethereum Chrome App. Checking out their most actively maintained [Node JS api](https://github.com/LedgerHQ/ledger-node-js-api/blob/77e3a1aba1f7ec7572b6752fb475aa0d3cb88efc/src/ledger-btc.js), there's the option to get the corresponding address for a wallet path, but that's it. You can't even do this from [MyEtherWallet](https://www.myetherwallet.com/), probably because the API isn't well exposed from the Ledger.

Their [ledger-wallet-api](https://github.com/LedgerHQ/ledger-wallet-api) provides hope—though it's untouched since October of 2016. They have an example page where you can query your xpub while connected to their Bitcoin Chrome App. Their Ethereum version was under development on the develop branch, but was apparently abandoned before xpub functionality was added.

Luckily this key hierarchy is shared between coins, regardless of the unfortunate decision to create separate applications for separate cryptos.


# Extract the xpub
1. KYP—Know Your Path. You need to know the BIP44 path of the xpub you're trying to generate. Generally, you want the xpub for the last hardened child in your particular path. For an account number *a<sub>n</sub>*, paths are: Bitcoin legacy <code>m/44'/0'/a<sub>n</sub>'/</code>; Bitcoin Segwit <code>m/49'/0'/a<sub>n</sub>'/</code>; Ethereum <code>m/44'/60'/a<sub>n</sub>'/</code>. Your account number is probably 0. For more details about path derivation, see [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki). For some alternatives about how software might have structured your path (if you can't find your coins), check out [this debate](https://github.com/ethereum/EIPs/issues/84).

2. Visit the [ledger-wallet-api demo page for Bitcoin](https://www.ledgerwallet.com/api/demo.html) & click "Launch Ledger app".

3. Unlock your ledger, paste your path next to the "Get xpubkey" button, then press it & you're good to go.
