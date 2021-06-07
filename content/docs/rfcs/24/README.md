---
slug: 24
title: 24/WAKU-VOTING
name: Waku v2 Voting
status: raw
editor: Szymon Szlachtowicz <szymon.s@ethworks.io>
---

This spec is a proposition for a voting protocol over Waku V2.

# Motivation

In open p2p protocol there is an issue with voting off-chain as there is much room for malicious peers to only include votes that support their case when submitting votes to chain.

Proposed solution is to agregate votes over waku and allow users to submit votes to smart contract that aren't already submitted.

# Smart contract

Voting should be finalized on chain so that the finished vote is immutable. Becouse of that, smart contract needs to be deployed. New smart contract will have to be able to start a vote and accept submited votes.

## Double voting

Smart contract should also keep a list of all votes so that no one can double vote.

# Voting

## Sending votes

Sending votes is simple every peer is able to send a message to Waku topic specific to given application: 
```
/{application-name}/{voting-room}
```

vote object that is sent over waku should contain information about: 

```ts
{
    sender: String //address of a sender
    vote: String //vote sent eg. 'yes' 'no'
    sntAmount: BigNumber //number of snt cast on vote
    sign: String // cryptographic sign of a transaction
    nonce: Number // number of votes cast from this address on current vote
    room: String // address of a voting room
}
```

## Aggregating votes

Every peer that is opening specific voting room will listen to votes sent over p2p network, and aggregate them for a single transaction to chain

## Submiting to chain

Every peer that has aggregated at least one vote will be able to send them to smart contract.

Smart contract needs to verify that all votes are valid (eg. all adress had enough SNT, all votes are correctly signed) and that votes aren't duplicated on smart contract.

## Finalizing 

Once given time window (block of given number was mined) has passed smart contract will no longer accept adding new votes and it will be possible to call smart contract with address of voting room and get back result of voting.

