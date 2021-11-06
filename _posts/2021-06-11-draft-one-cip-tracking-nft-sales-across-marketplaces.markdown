---
layout: post
title:  "Draft 1: CIP Tracking NFT sales across marketplaces"
author: Srdjan Stankovic
date:   2021-11-06 21:00:00 +0100
tags: [blockchain,cardano,cip,nft,marketplace]
image: "/assets/cardano.jpeg"
summary: "Proposal about standardising metadata for NFT token sales. This proposal would help in tracking NFT sales across marketplaces."
---

Discuss about this proposal on [Cardano Forum][cardano-forum].


## Intro

In this proposal I'd like to propose and discuss standard for structring transaction metadata for NFT sales.

## Motivation

While working on Cardano NFT Marketplace we've come to idea to track history of the NFT (token sales, etc.). Currently we have no way of tracking sales for token accross markets.

By standardising metadata for token sales we could show users history of that token, on what marketplaces it has been listed and sold before.

## Proposal

In order to track token sales across markets we'd need to include metadata about the token sale. By having standardised metadata structure we can ensure that we can always parse transaction metadata.

```json
{
  "765": {
    "<asset id>": {
      "marketplace": {
        "url": "<marketplace-url>",
        "name": "<marketplace-name>",
		"logo": "ipfs://<ipfs-hash>"
      },
	  "amount": "<amount-number>",
      "purchaseType": "<sale | auction>"
    }
  }
}
```

- `asset id` - ID of Aseet that has been sold (could be possibly asset fingerprint or something similar)
- `marketplace.url`  - URL of marketplace that token has been listed/sold on
- `marketplace.name` - Name of marketplace that token has been listed/sold on
- `marketplace.logo` - Logo of marketplace that token has been listed/sold on
- `amount` - Amount of tokens sold (This is possibly redudant data)
- `price`  - Total amount of lovelace that token(s) have been sold for (This is also possibly reduntant data)
- `purchaseType` - Provides info about the way token has been purchased, weither it's auction or sale.

[cardano-forum]: https://forum.cardano.org/t/cip-tracking-nft-sales-across-marketplaces/83536
