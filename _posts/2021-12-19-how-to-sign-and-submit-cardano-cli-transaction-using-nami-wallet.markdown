---
layout: post
title: "How to sign and submit cardano-cli transaction using Nami Wallet"
author: Srdjan Stankovic
date: 2021-12-24 10:00:00 +0100
tags:
  [
    blockchain,
    cardano,
    cardano-cli,
    nami-wallet,
    transaction,
    transation-builder,
  ]
image: "/assets/nami.png"
summary: "Bootstrap your Cardano dApp by signing and submiting transaction built by cardano-cli using Nami wallet."
---

## Intro

While working on integrating smart contracts with Nami Wallet I've faced many obsticles, most of them regarding `cardano-serialization-lib`. Most of the obsticles are due to lack of documentation and examples for this library, so we've decided to go other way around.

Building transactions using `cardano-cli` is pretty much easiest way to get started as there's a lot of examples and most of the developers working with Cardano ecosystem first get familiar with using `cardano-cli`.

And then when we finaly tought we've solved our issue, new ones appeard.

## Issues

Mayor issue that we and many others faced by trying to load transaction CBOR produced by `cardano-cli` to `Transaction` from `cardano-serialization-lib` is witness-set incompatibility. Non-signed transaction produced by `cardano-cli` has empty array for transaction witness-set while `cardano-serialization-lib` expects an empty map (as per design documents).

Here's an example of an unsigned transaction produced by `cardano-cli`:

```
[
   {...},// tx body
   [],   // empty witness-set
   [],   // datum
   [],   // redeemr
   true, // is-valid
   null  // metadata (no metadata in our case)
]
```

As you can see in the example, witness-set is returned as an empty array.

## Solution

The solution for this is really simple -- generate some address and sign the generated transaction using that address signing keys. You can sign transactions using and keys but I'd heavily advise not to use signing keys of some address that has or will receive funds in the future.

Once a transaction is signed using `cardano-cli` it should be fully assembled by the CLI and have this shape:

```
[
   {...}, // tx body
   {...}, // witness-set holding vkeys, datum, redeemer, etc.
   true,  // is valid
   null,  // metadata (no metadata in our case)
]
```

## Signing and submiting

Once we have the signed transaction using the `cardano-cli` we would need to replace signing keys in order to submit that transaction using Nami Wallet. In order to do that we first need to load the cbor, copy the transaction body and witness set, and empty the signing keys.

```javascript
// load tx cbor
const txCli = cardano.Transaction.from_bytes(Buffer.from(txCbor, "hex"));

// get tx body
const txBody = txCli.body();

// get tx witness-set
const witnessSet = txCli.witness_set();

// clear vkeys from witness-set
witnessSet.vkeys()?.free();
```

After clearing vkeys for the witness-set we can re-assemble the transaction using tx body and cleaned witness-set and set required signatures to our wallet address.

```javascript
// create required signers
const walletAddress = cardano.BaseAddress.from_address(decodedAddress);
const requiredSigners = cardano.Ed25519KeyHashes.new();
requiredSigners.add(walletAddress.payment_cred().to_keyhash());

// set newly created required signer to our tx body
txBody.set_required_signers(requiredSigners);

// re-assemble transaction
const tx = cardano.Transaction.new(txBody, witnessSet);
```

Finally when we have re-assembled the transaction, we can encode it, sign it using Nami Wallet and submit it.

```javascript
// encode tx
const encodedTx = Buffer.from(tx.to_bytes()).toString("hex");
// sign tx using nami wallet
const encodedTxVkeyWitnesses = await wallet.signTx(encodedTx, true);

// decode witness-set produced by signature
const txVkeyWitnesses = cardano.TransactionWitnessSet.from_bytes(
  Buffer.from(encodedTxVkeyWitnesses, "hex")
);

// set vkeys to our tx from decoded witness-set
witnessSet.set_vkeys(txVkeyWitnesses.vkeys());

// re-assemble signed transaction
const txSigned = cardano.Transaction.new(tx.body(), witnessSet);

// encode signed transaction
const encodedSignedTx = Buffer.from(txSigned.to_bytes()).toString("hex");

// submit the transaction
const txHash = await wallet.submitTx(encodedSignedTx);
```

## Full code example

Here's a full code snippet for signing and submiting the transaction.

```javascript

import type * as Cardano from "@emurgo/cardano-serialization-lib-browser";
import { Wallet } from "../../context/nami-wallet";
type CardanoT = typeof Cardano;

const decodeAddress = (cardano: CardanoT, encodedAddress: string): string =>
    cardano.Address.from_bytes(Buffer.from(encodedAddress, "hex")).to_bech32();

const getWalletAddress = async (
  cardano: CardanoT,
  wallet: Wallet
): Promise<WasmNamespace.Address> => {
  const encodedAddress = (await wallet?.getUsedAddresses())[0];
  const decodedAddress = cardano.Address.from_bech32(
    decodeAddress(cardano, encodedAddress)
  );

  return decodedAddress;
};

export const finalizeTx = async (
  cardano: CardanoT,
  wallet: Wallet, // Nami Wallet
  txCbor: string
) => {

  // Get address from Nami Wallet
  const decodedAddress: Cardano.Address = await getWalletAddress(cardano, wallet);

  /*
   * Load and sign transaction
   */
  const txCli = cardano.Transaction.from_bytes(Buffer.from(txCbor, "hex"));
  const txBody = txCli.body();

  const witnessSet = txCli.witness_set();

  witnessSet.vkeys()?.free();

  const walletAddress = cardano.BaseAddress.from_address(decodedAddress);
  const requiredSigners = cardano.Ed25519KeyHashes.new();
  // @ts-ignore
  requiredSigners.add(walletAddress.payment_cred().to_keyhash());
  txBody.set_required_signers(requiredSigners);

  const tx = cardano.Transaction.new(txBody, witnessSet);

  // encode tx and sign it using wallet
  const encodedTx = Buffer.from(tx.to_bytes()).toString("hex");
  const encodedTxVkeyWitnesses = await wallet.signTx(encodedTx, true);

  const txVkeyWitnesses = cardano.TransactionWitnessSet.from_bytes(
    Buffer.from(encodedTxVkeyWitnesses, "hex")
  );

  // @ts-ignore
  witnessSet.set_vkeys(txVkeyWitnesses.vkeys());

  const txSigned = cardano.Transaction.new(tx.body(), witnessSet);

  const encodedSignedTx = Buffer.from(txSigned.to_bytes()).toString("hex");

  const txHash = await wallet.submitTx(encodedSignedTx);
  return txHash;
};
```

## Summary

Creating transactions using `cardano-serialization-lib` can be frustrating at times, especially due to constant changes to library API. This approach can hopefully help you to bootstrap your dApp quickly, and later on down the road replace the `cardano-cli` with `cardano-serialization-lib`.
