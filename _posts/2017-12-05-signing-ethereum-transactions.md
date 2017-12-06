---
layout: post
title: Note on Signing Ethereum Transactions on Ledger Nano S with ethereumjs-tx
tags:
- Ethereum
- Crypto
- Software
---

I ran into some problems with [ethereumjs-tx](https://github.com/ethereumjs/ethereumjs-tx) when attempting to make use of [Ledger's Node API](https://github.com/LedgerHQ/ledger-node-js-api) for transaction signing. The Ledger API is pretty straightforward: it takes a BIP32 key path & a hex serialized transaction, and returns the signature data for the transaction.

```javascript
let eth = ... // create ledger eth object
eth.signTransaction_async("44'/60'/0'/0'/0", "e8018504e3b292008252089428ee52a8f3d6e5d15f8b131996950d7f296c7952872bd72a2487400080")
    .then(console.log).catch(console.log);
```

Since ethereumjs-tx takes care of structuring and [rlp encoding](https://github.com/ethereum/wiki/wiki/RLP) transactions, you'd imagine that a signature could be as easy as:

```javascript
const txnData = {
    nonce: web3.utils.toHex(/* the nonce */),
    gasPrice: web3.utils.toHex(26000000000),
    gasLimit: web3.utils.toHex(21000), // simple tx always consumes 21k gas
    to: /* some address */,
    value: web3.utils.toHex(web3.utils.toWei(/* some amount */)),
};
const txn = new tx(txnData);
eth.signTransaction_async(keyPath, txn.serialize().toString('hex'));
.then(signed => {
    // Use the signature data
}).catch(err => console.log("Failed..."));
```

## Formatting the transaction
The hex string to be passed to the ledger actually requires some doctoring. Because of [EIP155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md) empty signature data is
included in the transaction, and one field is replaced with the chain code. So instead of *(v, r, s)* it's *(chainCode, 0, 0)*, but it must be part of the hex data signed by the ledger.
ethereumjs-tx ought to take care of this when it serializes the unsigned transaction, but it doesn't unless you sign the transaction via its API. Luckily the changes are simple to make:

```javascript
const txn = new tx(/* the data */);
txn.v = 1;
txn.r = 0;
txn.s = 0;
const hexSerialized = txn.serialize().toString('hex');
```

## Using the signature data
Once the signature data are reported by the ledger, they need to be formatted properly & injected back into the transaction object.

```javascript
// Create the transaction & hex-serialize it
const txn = new tx(/* the data */);
txn.v = 1;
txn.r = 0;
txn.s = 0;
const hexSerialized = txn.serialize().toString('hex');

let eth = ... // create ledger eth object
eth.signTransaction_async(/* some key path */, hexSerialized)
.then(sigData => {
    // Format it to be inserted back into the txn object
    const formattedData = {
        v: new Buffer(sigData.v, 'hex').slice(0, 1),
        r: new Buffer(sigData.r, 'hex').slice(0, 32),
        s: new Buffer(sigData.s, 'hex').slice(0, 32),
    };

    // Inject the signature data manually
    Object.assign(txn, sigData);

    // Now you can do whatever you want with the txn...
    web3.eth.sendSignedTransaction('0x' + txn.serialize().toString('hex')).then(res => {
        alert("Transaction broadcast! " + JSON.stringify(res));
    }).catch(err => {
        alert("Transaction failed to broadcast! " + err);
    });
}).catch(console.log);
```
