# AEX X

```
AEX: Maybe 12
Title: Verifiable off-chain state for accounts
Author: Michal Zajda <michal@sennui.com>
License: BSD-2-Clause
Discussions-To: https://forum.aeternity.com/c/aepplications/improvement-proposals
Status: Draft
Type: Interface
Created: 2019-09-24
```

## Simple Summary

This expansion defines way of orchestrating updates to users data
across 3rd party systems without bloating blockchain state.

## Abstract

Expansion proposes lightweight protocol for coordinating updates
in arbitrary off-chain storage. It is a way of communicating user with 3rd
party services.

It uses `zero spend` transactions with payload, spent to user's own account.
This document standardizes communication with 3rd party subscribers, by proposing
`spend` payload schema. Blockchain serializes transactions and enables
consistency. `Spend` signature will prove origin of the data.
`Payload` can carry arbitrary data message understood by participating parties.
The message may have off-chain side effects in 3rd party systems.

Expansion uses the most basic transaction (`spend`) to reduce API access risks and
improve insight in raw structure. Expansion proposes options for further
reasoning about data completeness.

The expansion balances need for extending on-chain protocol and reduces risk
of on-chain state growth. Middleware software indexing data from blockchain is
especially well suited to use `zero spend`.


Example use-cases cover: Proof of Existence for large media files,
Identity services, Cross-account user's state management like reverse-lookup config.


## Motivation

Designers of blockchain systems try to limit amount of the on-chain data.
It is caused by limits of resources that can be provided within the
unit of a single node. Scarce resources in blockchains are: computational power
and fast access memory. This design goal is incentivised with high size
and storage gas prices.

Naturally, it limits all use-cases that could benefit from having global,
consistent, auditable, signed data storage for user's data, available for 3rd
party systems.

This expansion aids such use-cases. It leverages possibility of using
transaction payload to orchestrate multiple 3rd party updates of user's off
chain state. It will affect only raw chain data structure and limit on-chain
state growth.

3rd party systems can listen to the chain and when they match their user's
update, they can apply it in local systems. It is especially easy when
subscribed to blockchain middleware. Also blockchain middleware may maintain
such state per user.


## Specification

### Definitions
|Term | Definition |
|:---|:---|
|Account| an account in Aeternity blockchain|
|User| entity that has 1+ accounts in Aeternity and/or 3rd party systems|
|Spend| standalone spend transaction in Aeternity blockchain|
|Zero Spend| spend transaction that transfers zero tokens|
|Message| arbitrary data part of spend payload|


### Transaction schema

According to [Aeternity protocol](https://github.com/aeternity/protocol/blob/master/serializations.md#spend-transaction)
spend has following specification:

```
[ <sender>    :: id()
, <recipient> :: id()
, <amount>    :: int()
, <fee>       :: int()
, <ttl>       :: int()
, <nonce>     :: int()
, <payload>   :: binary()
]
```

Following conditions MUST be met to trigger communication proposed in expansion:
1. `<sender>` MUST be equal to `<receipient>`
2. `<amount>` MUST be equal to `0`
3. `<payload>` MUST be non empty

### Payload specification

Payload size is limited by size of a microblock. As of Lima hardfork, gas limit
translates to about 20 kilobytes of data. Assuming a miner will accept one
spend transaction consuming whole gas, this will be the max value of payload.
It is further reduced by headers and spend transaction envelope.

Payload is JSON document.

Payload is Gzipped and set to payload field in spend transaction.

AEX-X payload has one required field: `data`

* `data` is arbitrary data understood by 3rd party

* `dest` is used to cheaply determine if the `message` is send to given subscriber.

`dest` reduces overhead of trying to decode every message, but it is not reserved value
and may cause conflicts.


### Recommendation

It is recommended to use cryptographically secure data structure to track hashes
of data and reason about completeness of obtained data.

Root of the tree should be added to custom `data` field.

Root mismatch may indicate lack of data due to chain fork or
excess of data in case of `dest` collision.

### JSON payload schema

```
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "Payload",
  "description": "Unified spend transaction payload",
  "type": "object",
  "properties": {
    "data": {
      "description": "Arbitrary data",
      "type": "string"
    },
    "dest": {
      "description": "Tag to use by 3rd party, to distinguish destination",
      "type": "string"
    }
  },
  "required": ["data"]
}
```

### Example

Payload:

```
{
  "data": "Hello Joe."
  "dest": "Mike"
}
```

Gzipped payload:

```
<<"31,139,8,0,0,0,0,0,0,19,243,72,205,201,201,87,240,202,79,213,3,0,78,76,92">>
```

Base64 payload encoding if needed:

```
"H4sIAAAAAAAAE/NIzcnJV/DKT9UDAE5MXPgKAAAAZ71MJg=="
```

Spend:
```
{
  "amount":0,
  "fee":20000000000000,
  "payload": "H4sIAAAAAAAAE/NIzcnJV/DKT9UDAE5MXPgKAAAAZ71MJg==",
  "recipient_id":"ak_BKSzaEuzHRYokFC1YK4KgHj6h9rnjAmAjxDQF43ZCopy8RWdZ",
  "sender_id":"ak_BKSzaEuzHRYokFC1YK4KgHj6h9rnjAmAjxDQF43ZCopy8RWdZ"
}
```

## Rationale

We should optimize for size of the payload.

We use Gzip + JSON as it is proven to be about the same size as available serialization formats (e.g. msgpack, bson)
and has advantage of being human readable after decompression. [Reference needed]

Root hash of the data is not enforced as multiple 3rd party can listen to the same account and may be interested
only in the subset of emitted data.

## Use Cases

1. Naming reverse-lookup. Multiple Names points to the same address in the blockchain.
   When a 3rd party app aggregates the names it doesn't know which name to use.

   If it is a game, the gaming service would like to use the same name everywhere.
   If a user uses the same account in multiple games, he may expect that games
   translate his account to the same label.

   User may set up one as a main one using `Zero spend`.
   User adds Message prepared by 3rd party and sends `Zero spend`.
   3rd party service sets up local state based on `Zero spend`.

   If there is another 3rd party service or another game, it set up the state
   in consistent way, as `Zero spend` is confirmed transaction.

   User is represented everywhere with the same label.
   Blockchain in-memory state is not bloated with another data which holds reverse lookup.
   Raw chain structure lives on hard drive with is orders of magnitude cheaper.

2. Identity services
   User creates an account linked to his ID via 3rd party service.
   `Zero spend` creates unique id representing him in ecosystem of services
   who trust original 3rd party.

3. Proof of Existence:
   User issues `Zero spend` with hash of a binary artifact. Interested services
   listen and extend user state with the hash. In case of dispute, they exchange
   data and compare hashes. Example:
   * car rental

   Another vendors may use this information. Examples:
   * hash of unique object in the game
   * hash of the song that is playing


## Implementation

No specific implementation. AEX uses any wallet application.
For HTTP operation base64 encoding is required.
