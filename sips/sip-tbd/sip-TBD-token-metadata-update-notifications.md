# Preamble

SIP Number: TBD

Title: Notifications for Token Metadata Updates

Author: Rafael Cárdenas (rafael@hiro.so), Matthew Little (matt@hiro.so)

Consideration: Technical

Type: Standard

Status: Draft

Created: 17 May 2022

License: GPL-3.0

Sign-off: TBD

Layer: Traits

# Abstract

As the use of tokens (fungible and non-fungible) has grown in popularity, Stacks developers have
found novel ways to define and use metadata to describe them. This rich data is commonly cached and
indexed in applications such as marketplaces, statistics aggregators, and developer tools like the
[Stacks Blockchain API](https://github.com/hirosystems/stacks-blockchain-api) for future use.

Occasionally, however, this metadata needs to change for a number of reasons: artwork reveals, media
storage migrations, branding updates, etc. As of today, these changes do not have a standardized way
of being propagated through the network so indexers can refresh their cache, so the display of stale
metadata is a very common problem.

This SIP aims to define a simple mechanism for developers to notify the Stacks network when metadata
for a token has changed, so interested parties can refresh their cache and display up-to-date
information in their applications.

# Introduction

Smart contracts which declare NFTs and FTs conform to a standard set of traits defined in
[SIP-009](../sip-009/sip-009-nft-standard.md) and
[SIP-010](../sip-010/sip-010-fungible-token-standard.md) respectively. One of those traits is
`get-token-uri`, which should return a valid URI string that resolves to the token's metadata
(usually a JSON file).

As an example, when we perform a `get-token-uri` function call against the
[`SPSCWDV3RKV5ZRN1FQD84YE1NQFEDJ9R1F4DYQ11.newyorkcitycoin-token-v2`](https://explorer.stacks.co/txid/0x969192220b1c478ef9d18d1cd413d7c79fe02937a9b33af63d441bd5519d1715?chain=mainnet)
contract (at the time of writing), we get the string value for
https://cdn.citycoins.co/metadata/newyorkcitycoin.json which, when resolved, returns:

```json
{
  "name": "NewYorkCityCoin",
  "description": "A CityCoin for New York City, ticker is NYC, Stack it to earn Stacks (STX)",
  "image": "https://cdn.citycoins.co/logos/newyorkcitycoin.png"
}
```

Additionally, `.newyorkcitycoin-token-v2` (as other token contracts) also includes a way for owners to
change this URI via a `var-set` function call, like so:

```clarity
(define-data-var tokenUri (optional (string-utf8 256)) (some u"https://cdn.citycoins.co/metadata/newyorkcitycoin.json"))

;; set token URI to new value, only accessible by Auth
(define-public (set-token-uri (newUri (optional (string-utf8 256))))
  (begin
    (asserts! (is-authorized-auth) ERR_UNAUTHORIZED)
    (ok (var-set tokenUri newUri))
  )
)
```

This is a very flexible setup that, for instance, allows `.newyorkcitycoin-token-v2` contract
administrators to tweak the token's description or change its logo when they see fit. Nevertheless,
it creates a complex problem for metadata indexers which now need to figure out if (and when) they
should re-index token contracts to avoid displaying stale metadata in their applications.


## Metadata staleness

Within the Stacks ecosystem, there are a number of applications and tools that need to index token
metadata. Some common examples are an NFT marketplace which needs to display a token's artwork
for users to view, and a [blockchain API](https://github.com/hirosystems/stacks-blockchain-api)
which needs to serve FT metadata to display account balances correctly in its responses.

In order to achieve this, developers usually run and maintain a background process that listens for
new SIP-009/SIP-010 compliant contracts deployed to the blockchain (through the Stacks node RPC
interface) so they can immediately call on its metadata and save the results to a local database or
file storage bucket. While this works correctly for new contracts, it is insufficient for old
ones that may change their metadata at any point in the future and thus cause staleness.

As a short-term solution, most indexers currently resort to a cron-like periodic refresh of all
tracked contracts which guarantees a certain level of data freshness. While this may work for
individual applications, though, it does not provide a consistent experience for Stacks users that
may interact with different metadata-aware applications that have different refresh periods.
Additionally, this method creates inefficiencies like unnecessary network traffic, extra strain on
public Stacks nodes, etc.

## Metadata update notifications

To solve this problem, smart contract developers need a way to notify network participants that they
have made changes to the metadata so any indexers may then perform a refresh just for that contract.
Furthermore, this notification should create a persistent record in the Stacks blockchain so
indexers that may be unavailable when the notification is broadcasted can still receive the message
and perform the refresh when they come back online.

This SIP aims to establish a simple way of achieving this goal using contract tools already
available to developers.

# Specification

The proposed broadcast mechanism for token metadata update notifications makes use of the [`print`
Clarity language function](https://docs.stacks.co/write-smart-contracts/language-functions#print).

When `print` is used in a smart contract, an event of type `contract_event` is emitted which contains its output inside the `value` key, like so:

```json
{
  "type": "contract_event",
  "contract_event": {
    "contract_identifier": "<emitter contract>",
    "topic": "print",
    "value": "<print output>"
  }
}
```

This event is then attached to a transaction object which is broadcasted to subscribed RPC listeners
(like metadata indexers) through Stacks nodes when the same transaction is included in a block or microblock.

Taking advantage
of this fact, this SIP simply proposes a short message structure (similar to a notification payload) that would signal to indexers
that a contract needs its metadata refreshed. The proposed notification payload would be a tuple with the following structure:
```clarity
{ notification: "token-metadata-update", payload: { token-class: "ft", contract-id: contract }}
```
* The `notification` key should always contain the string `"token-metadata-update"`
* The `payload` key should contain another tuple with the following keys:
    * `token-class`, which would be `"ft"` or `"nft"` depending on the class of token being refreshed (and could later be expanded to include `"sft"` for semi-fungible tokens)
    * `contract-id`, which would contain the principal for the contract which needs its token metadata refreshed

Following this structure, contracts could implement a very simple function that allows developers to emit this notification, for example:

```clarity
(define-public (metadata-update-notify)
  (ok (print { notification: "token-metadata-update", payload: { token-class: "nft", contract-id: (as-contract tx-sender) }})))
```

When an indexer

# Backwards compatibility

Developers who need to update metadata for contracts that were deployed before this SIP is activated could use a mechanism similar to the one described in [Reference Implementations](#reference-implementations) to notify indexers of any metadata changes.

# Activation

TBD

# Reference implementations

As part of this SIP, a contract will be deployed similar to send-many that would simplify this task for users who wish to broadcast updates for old contracts or for many contracts at the same time.

```clarity
;; send-many
(define-public (metadata-update (ustx uint) (to principal) (memo (buff 34)))
 (let ((transfer-ok (try! (stx-transfer? ustx tx-sender to))))
   (print memo)
   (ok transfer-ok)))

(define-private (send-stx (recipient { to: principal, ustx: uint, memo: (buff 34) }))
  (send-stx-with-memo
     (get ustx recipient)
     (get to recipient)
     (get memo recipient)))

(define-private (check-err (result (response bool uint))
                           (prior (response bool uint)))
  (match prior ok-value result
               err-value (err err-value)))

(define-public (metadata-update-many (recipients (list 200 { to: principal })))
  (fold check-err
    (map send-stx recipients)
    (ok true)))
```

The Stacks Blockchain API will also add compatibility for this standard.