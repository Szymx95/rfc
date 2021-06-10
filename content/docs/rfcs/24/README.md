---
slug: 24
title: 24/STATUS-CURATION
name: Status Community Directory Curation Voting using Waku v2
status: raw
category: application
editor: Szymon Szlachtowicz <szymon.s@ethworks.io>
---

This spec is a proposition for a voting protocol over Waku V2.

# Motivation

In open p2p protocol there is an issue with voting off-chain as there is much room for malicious peers to only include votes that support their case when submitting votes to chain.

Proposed solution is to agregate votes over waku and allow users to submit votes to smart contract that aren't already submitted.

# Smart contract

Voting should be finalized on chain so that the finished vote is immutable. 
Becouse of that, smart contract needs to be deployed. 
When votes are submitted smart contract has to verify what votes are properly signed and that sender has correct amount of SNT.
When Vote is verified the amount of SNT voted on specific room by specific sender is saved on chain.

## Double voting

Smart contract should also keep a list of all signatures so that no one can send the same vote twice.
Another possibility is to allow each sender to only vote once.

## Initializing Vote

When someone wants to initialize vote he has to send a transaction to smart contract that will create a new voting room.
When initializing a user has to specify type of vote (Addition, Deletion),amount of his initial SNT to submit and public key of community under vote.
Smart contract will return a ID which is identifier of voting room.
Also there will be function on Smart Contract that when given community public key it will return voting room ID or undefined if community isn't under vote.

# Voting

## Sending votes

Sending votes is simple every peer is able to send a message to Waku topic specific to given application: 
```
/status-community-directory-curation-vote/1/{voting-room-id}/proto
```

vote object that is sent over waku should contain information about: 

```ts
{
    sender: String // address of the sender
    vote: String // vote sent eg. 'yes' 'no'
    sntAmount: BigNumber //number of snt cast on vote
    sign: String // cryptographic signature of a transaction (signed fields: sender,vote,sntAmount,nonce,roomID)
    nonce: Number // number of votes cast from this address on current vote (only if we allow multiple votes from the same sender)
    roomId: Number // ID of voting room
}
```

## Aggregating votes

Every peer that is opening specific voting room will listen to votes sent over p2p network, and aggregate them for a single transaction to chain.

## Submiting to chain

Every peer that has aggregated at least one vote will be able to send them to smart contract.
When someone votes he will aggregate his own vote and will be able to immediately send it.

Peer doesn't need to vote to be able to submit the votes to the chain.

Smart contract needs to verify that all votes are valid (eg. all senders had enough SNT, all votes are correctly signed) and that votes aren't duplicated on smart contract.

## Finalizing 

Once the vote deadline has expired, the smart contract will not accept votes anymore.
Also directory will be updated according to vote results (community added to directory, removed etc.)