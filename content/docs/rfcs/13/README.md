---
slug: 13
title: 13/WAKU2-STORE
name: Waku v2 Store
status: draft
tags: waku-core
editor: Sanaz Taheri <sanaz@status.im>
contributors:
  - Dean Eigenmann <dean@status.im>
  - Oskar Thorén <oskar@status.im>
---

This specification explains the Waku `13/WAKU2-STORE` protocol which enables querying of messages received through relay protocol and stored by other nodes. 
It also supports pagination for more efficient querying of historical messages. 

**Protocol identifier***: `/vac/waku/store/2.0.0-beta3`

# Design Requirements
Nodes willing to provide storage service using `13/WAKU2-STORE` protocol SHOULD provide a complete and full view of message history.
As such, they are required to be *highly available* and in specific have a *high uptime* to consistently receive and store network messages. 
The high uptime requirement makes sure that no message is missed out hence a complete and intact view of the message history is delivered to the querying nodes.
Nevertheless, in case that storage provider nodes cannot afford high availability, the querying nodes may retrieve the historical messages from multiple sources to achieve a full and intact view of the past.

# Security Consideration

The main security consideration to take into account while using `13/WAKU2-STORE` is that a querying node have to reveal their content filters of interest to the queried node, hence potentially compromising their privacy.

## Terminology
The term Personally identifiable information (PII) refers to any piece of data that can be used to uniquely identify a user. 
For example, the signature verification key, and the hash of one's static IP address are unique for each user and hence count as PII.

# Adversarial Model
Any peer running the `13/WAKU2-STORE` protocol, i.e. both the querying node and the queried node, are considered as an adversary. 
Furthermore, we currently consider the adversary as a passive entity that attempts to collect information from other peers to conduct an attack but it does so without violating protocol definitions and instructions. 
As we evolve the protocol, further adversarial models will be considered.
For example, under the passive adversarial model, no malicious node hides or lies about the history of messages as it is against the description of the `13/WAKU2-STORE` protocol. 

The following are not considered as part of the adversarial model:
- An adversary with a global view of all the peers and their connections.
- An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). 
  In specific, the communication channels are assumed to be secure.


# Wire Specification
Peers communicate with each other using a request / response API. 
The messages sent are Protobuf RPC messages which are implemented using [protocol buffers v3](https://developers.google.com/protocol-buffers/).
The followings are the specifications of the Protobuf messages. 

## Payloads

```protobuf
syntax = "proto3";

message Index {
  bytes digest = 1;
  double receiverTime = 2;
  double senderTime = 3;
}

message PagingInfo {
  uint64 pageSize = 1;
  Index cursor = 2;
  enum Direction {
    BACKWARD = 0;
    FORWARD = 1;
  }
  Direction direction = 3;
}

message ContentFilter {
  string contentTopic = 1;
}

message HistoryQuery {
  // the first field is reserved for future use
  string pubsubtopic = 2;
  repeated ContentFilter contentFilters = 3;
  PagingInfo pagingInfo = 4;
}

message HistoryResponse {
  // the first field is reserved for future use
  repeated WakuMessage messages = 2;
  PagingInfo pagingInfo = 3;
}

message HistoryRPC {
  string request_id = 1;
  HistoryQuery query = 2;
  HistoryResponse response = 3;
}
```

### Index

To perform pagination, each `WakuMessage` stored at a node running the `13/WAKU2-STORE` protocol is associated with a unique `Index` that encapsulates the following parts. 
- `digest`:  a sequence of bytes representing the SHA256 hash of a `WakuMessage`.
  The hash is computed over the concatenation of `contentTopic` and `payload` fields of a `WakuMessage` (see [14/WAKU2-MESSAGE](/spec/14)).
- `receiverTime`: the UNIX time at which the waku message is received by the receiving node.
- `senderTime`: the UNIX time at which the waku message is generated by its sender.

### PagingInfo

`PagingInfo` holds the information required for pagination. It consists of the following components.
- `pageSize`: A positive integer indicating the number of queried `WakuMessage`s in a `HistoryQuery` (or retrieved  `WakuMessage`s in a `HistoryResponse`).
- `cursor`: holds the `Index` of a `WakuMessage`.
- `direction`: indicates the direction of paging which can be either `FORWARD` or `BACKWARD`.

### ContentFilter
`ContentFilter` carries the information required for filtering historical messages. 
- `contentTopic` represents the content topic of the queried historical Waku messages.
  This field maps to the `contentTopic` field of the [14/WAKU2-MESSAGE](/spec/14).
  
### HistoryQuery

RPC call to query historical messages.

- The `pubsubTopic` field MUST indicate the pubsub topic of the historical messages to be retrieved. 
  This field denotes the pubsub topic on which waku messages are published.
  This field maps to `topicIDs` field of `Message` in [`11/WAKU2-RELAY`](/spec/11).
  Leaving this field empty means no filter on the pubsub topic of message history is requested.
  This field SHOULD be left empty in order to retrieve the historical waku messages regardless of the pubsub topics on which they are published.
- The `contentFilters` field MUST indicate the list of content filters based on which the historical messages are to be retrieved.
  Leaving this field empty means no filter on the content topic of message history is required.
  This field SHOULD be left empty in order to retrieve historical waku messages regardless of their content topics.
- `PagingInfo` holds the information required for pagination.  Its `pageSize` field indicates the number of  `WakuMessage`s to be included in the corresponding `HistoryResponse`. If the `pageSize` is zero then no pagination is required. If the `pageSize` exceeds a threshold then the threshold value shall be used instead. In the forward pagination request, the `messages` field of the `HistoryResponse` shall contain at maximum the `pageSize` amount of waku messages whose `Index` values are larger than the given `cursor` (and vise versa for the backward pagination). Note that the `cursor` of a `HistoryQuery` may be empty (e.g., for the initial query), as such, and depending on whether the  `direction` is `BACKWARD` or `FORWARD`  the last or the first `pageSize` waku messages shall be returned, respectively.
The queried node MUST sort the `WakuMessage`s based on their `Index`, where the `senderTime` constitutes the most significant part and the `digest` comes next, and then perform pagination on the sorted result. 
As such, the retrieved page contains an ordered list of `WakuMessage`s from the oldest message to the most recent one.

Alternatively, the `receiverTime` (instead of `senderTime` ) MAY be used to sort `WakuMessage`s during the paging process. 
However, we RECOMMEND the use of the `senderTime` for sorting as it is invariant and consistent across all the nodes.
This has the benefit of `cursor` reusability i.e., a `cursor` obtained from one node can be consistently used to query from another node.
However, this `cursor` reusability  does not hold when the `receiverTime` is utilized as the receiver time is affected by the network delay and nodes' clock asynchrony.

### HistoryResponse

RPC call to respond to a HistoryQuery call.
- The `messages` field MUST contain the messages found, these are [`WakuMessage`] types as defined in the corresponding [specification](./waku-message.md).
- `PagingInfo`  holds the paging information based on which the querying node can resume its further history queries. 
  The `pageSize` indicates the number of returned waku messages (i.e., the number of messages included in the `messages` field of `HistoryResponse`). 
  The `direction` is the same direction as in the corresponding `HistoryQuery`. 
  In the forward pagination, the `cursor` holds the `Index` of the last message in the `HistoryResponse` `messages` (and the first message in the backward paging). 
  The requester shall embed the returned  `cursor` inside its next `HistoryQuery` to retrieve the next page of the waku messages.  
  The  `cursor` obtained from one node SHOULD NOT be used in a request to another node because the result MAY be different.

# Future Work

- **Anonymous query**: This feature guarantees that nodes can anonymously query historical messages from other nodes i.e., without disclosing the exact topics of waku messages they are interested in.  
As such, no adversary in the `13/WAKU2-STORE` protocol would be able to learn which peer is interested in which content filters i.e., content topics of waku message. 
The current version of the `13/WAKU2-STORE` protocol does not provide anonymity for historical queries as the querying node needs to directly connect to another node in the `13/WAKU2-STORE` protocol and explicitly disclose the content filters of its interest to retrieve the corresponding messages. 
However, one can consider preserving anonymity through one of the following ways: 
  - By hiding the source of the request i.e., anonymous communication. That is the querying node shall hide all its PII in its history request e.g., its IP address.
  This can happen by the utilization of a proxy server or by using Tor. 
  Note that the current structure of historical requests does not embody any piece of PII, otherwise, such data fields must be treated carefully to achieve query anonymity. 
  <!-- TODO: if nodes have to disclose their PeerIDs (e.g., for authentication purposes) when connecting to other nodes in the store protocol, then Tor does not preserve anonymity since it only helps in hiding the IP. So, the PeerId usage in switches must be investigated further. Depending on how PeerId is used, one may be able to link between a querying node and its queried topics despite hiding the IP address--> 
  - By deploying secure 2-party computations in which the querying node obtains the historical messages of a certain topic whereas the queried node learns nothing about the query. 
  Examples of such 2PC protocols are secure one-way Private Set Intersections (PSI). 
  <!-- TODO: add a reference for PSIs? --> <!-- TODO: more techniques to be included --> 
<!-- TODO: Censorship resistant: this is about a node that hides the historical messages from other nodes. This attack is not included in the specs since it does not fit the passive adversarial model (the attacker needs to deviate from the store protocol).-->

- **Robust and verifiable timestamps**: Messages timestamp is a way to show that the message existed prior to some point in time. However, the lack of timestamp verifiability can create room for a range of attacks, including injecting messages with invalid timestamps pointing to the far future.   
To better understand the attack, consider a store node whose current clock shows `2021-01-01 00:00:30` (and assume all the other nodes have a synchronized clocks +-20seconds).
The store node already has a list of messages `(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20)` that are sorted based on their timestamp.  
An attacker sends a message with an arbitrary large timestamp e.g., 10 hours ahead of the correct clock `(m',2021-01-01 10:00:30)`. 
The store node places `m'` at the end of the list `(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20), (m',2021-01-01 10:00:30)`. 
Now another message arrives with a valid timestamp e.g., `(m11, 2021-01-01 00:00:45)`. However, since its timestamp precedes the malicious message `m'`, it gets placed before `m'` in the list i.e.,  `(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20), (m11, 2021-01-01 00:00:45), (m',2021-01-01 10:00:30)`.
In fact, for the next 10 hours, `m'` will always be considered as the most recent message and served as the last message to the querying nodes irrespective of how many other messages arrive afterward. 

  A robust and verifiable timestamp allows the receiver of a message to verify that a message has been generated prior to the claimed timestamp. 
One solution is the use of [open timestamps](https://opentimestamps.org/) e.g.,  block height in Blockchain-based timestamps. 
That is, messages contain the most recent block height perceived by their senders at the time of message generation. 
This proves accuracy within a range of minutes (e.g., in Bitcoin blockchain) or seconds (e.g., in Ethereum 2.0) from the time of origination. 

# References
1. [Open timestamps](https://opentimestamps.org/) 
   
# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).



