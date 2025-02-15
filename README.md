# P2P timestamping guardrails for DID:PLC

## tl;dr

The directory signs the updates it receives. Anyone can run a directory. You can switch directory with a DID update. A directory can sign merkle trees of messages it signed.

We use a blockchain for the following limited purposes: Forcing through updates the directory won't sign, rotating the keys the directory signs with, and timestamping signed merkle roots for reorg-proofing. The reorg-proofing timestamps have no fixed schedule and anyone can create them at any time.

## Motivation

The centralized DID:PLC server model is working efficiently and economically. However, it breaks down if the server becomes malicious or broken.

This proposal attempts to mitigate or eliminate the problems that occur if it becomes malicious or broken, without compromising the functionality and low cost that users currently enjoy while it is running correctly.

## The DID:PLC model

Conceptually the DID:PLC server performs two distinct functions:

### Timestamping

Users sign their own DID update operations, but the validity or invalidity of an operation can depend on what time it was sent in relation to other operations on the same DID. The DID:PLC server registers the time at which it received the operation and uses this to validate or invalidate each operation. Any other user who possessed the same updates that the DID:PLC server did, and also knew at what time they were received, could unambiguously reach the same conclusion about the current state of the DID.

### Serving queries

Clients need to get their data about signed queries from somewhere. The DID:PLC server keeps a record of all updates, and serves the client either the current state of a DID or the record of updates to it (the audit log).

## Problems with the current DID:PLC model

The single source of authority about DID:PLC updates is the central DID:PLC server which serves updates over TLS. A malicious or broken DID:PLC could engage in:

### Censorship

The server can maliciously refuse to accept updates. 

It can also refuse to tell users and mirrors which updates it has accepted, in which case it will not be possible to run an honest mirror, so users cannot get information from any other source even if there is one they trust that was online when the updates were made.

This differs from the situation elsewhere in atproto, where although users may rely on a semi-trusted server such as a relay or appview to provide them with data, they can always get the data from somewhere else if the server misbehaves and verify its accuracy.

### Malicious reorgs

The server can lie about the relative times at which it received messages.

       time t           time t+1           time t+2            time t+3

    Genesis[Key 1] --------------------> Update1[Key 2] -------------------------->
                   |                    (timestamped t2)
                   |
                   -------------------------------------> Update1b[Attacker key]-->
                                                         (falsely timestamped t1)

Exploiting the ability to lie about timestamps requires possession of a key that would normally only be held by the user or someone to whom they delegate. However, we can imagine two plausible cases where the attacker possesses such a key. In the first, the user rotates from an old rotation key to a new one, and does not secure the old key. In the second, the user initially creates an account with a third-party, for example bsky.social, but subsequently migrates their account elsewhere to avoid needing to trust the third-party. In both cases the user reasonably expects that their old key can no longer be used to manipulate their account, but this assumption is not guaranteed by the current design of the DID:PLC system.

### Reliability / Availability

The server may simply start performing badly, in which case it would be better if clients could switch to an alternative.

## Our design

### Serving queries

We assume users will mostly continue to use a semi-trusted server for serving queries, which may or may not be the same server that timestamps their updates. These servers will continue to serve updates for all users (but could be sharded). The rest of this document will refer to any user trying to get DID:PLC entries as a "client", but in practice it is likely that the "client" is a semi-trusted server, and the end user queries that.

### Signing messages

When a timestamper timestamps a message, it also signs the `CID` of the update and the `createdAt` field. Each entry in the audit log will then contain not only the user's signature on the update, but also the timestamper's signature on the record accepting the update.

Clients must consider updates invalid unless they have been either signed by the timestamper, or timestamped on the blockchain (see below).

### Managing timestamper keys

The timestamper must maintain a [did:ethr](https://github.com/decentralized-identity/ethr-did-resolver/blob/master/doc/did-method-spec.md) record. The public key it signs with must be a valid `verificationMethod` at the time specified in the `createdAt` field, of type `EcdsaSecp256k1VerificationKey2019`.

### Changing timestampers

We add an additional field to the DID, `timestamperDid`. A user can update this field in their DID record in the same way they would update any other field in the DID record.

Entries with no `timestamperDid` specified are assumed to be using the DID representing the preexisting DID:PLC service.

### Forcing updates with P2P timestamping

If the timestamper is functioning correctly, as the DID:PLC directory has to date, a user can update their DID record with the timestamper as they do now. However, the user also has the option to send their update to the blockchain. In normal conditions this is slower and more expensive than sending it to their timestamper, so normally it would only be done if the timestamper is unavailable or refuses to accept their update.

Clients should read messages from the blockchain and include them as if they have been signed by the timestamper.

### Creating signed merkle trees

Timestampers should arrange snapshots of the updates they signed in a merkle tree and sign the root of the tree. At the timestamper's discretion this may be done immediately after every update, or only applied to periodic snapshots. They should share the content of this tree so that any user can verify the state of their DID record in any given tree.

Note that the failure to publish these signed merkle roots does not invalidate their updates, which have been individually signed.

### Timestamping signed merkle roots

Anyone may send a blockchain transaction to make a p2p timestamp of any signed merkle root at any time. Publishing the signed merkle root on the blockchain will allow a user in possession of the relevant part of the content of the tree to later prove that the timestamper had timestamped a given update at or before the time it was recorded on the blockchain. The blockchain timestamping of signed merkle roots is intended for reorg protection (see below) and no other purpose.

### Reorg protection

Given honest, correctly-functioning timestampers, anyone in receipt of all the signed messages for a DID can tell unambiguously what the state of the DID should be. This includes cases where an update was reverted by signing with a higher-priority rotation key, as the reversion rules can be applied unambiguously as long as the messages have been correctly timestamped.

However, as we discussed earlier, a malicious or broken timestamper may sign messages that purport to have been received earlier than they really were. This can result in conflicting versions of the DID. Clients need to be able to choose between these conflicting versions.

We propose that if a timestamper timestamps a version of the DID that conflicts with a version that has already been included in a signed merkle root timestamped on the blockchain, a client must honour the earlier blockchain-timestamped version. In the event of a conflict, a blockchain-timestamped update must always be considered to have been timestamped earlier than an update that has not been blockchain-timestamped.

Note that since both the creation and signing of the merkle root and its publication on the blockchain are optional, and the sharing of the data that would allow the construction of proofs against the merkle root is not enforced, a user can only be confident that their update is secure against a fraudulent reorg by the timestamper if the following conditions apply:

1. Their update is in a merkle root signed by the timestamper and published on the blockchain
2. They have the necessary intermediate tree node data to connect their update to that signed merkle root

In the scenario below, someone sends Update1 to the blockchain for a P2P timestamp at time t+3. Clients will reject Update1b sent by the malicious timestamper at time t+4 because they will see that it is in conflict with the version timestamped at time t+3. Had the malicious timestamper created Update1b after time t+2 and sent it to the blockchain before time t+3, their malicious update would have prevailed.

       time t         time t+1        time t+2          time t+3          time t+4

    Genesis[Key 1] --------------> Update1[Key 2] ------------------------------------------>
                   |              (timestamped t2)
                   |                               (blockchain timestamp)
                   |
                   -----------------------------------------------> Update1b[Attacker key]-->
                                                                   (falsely timestamped t1)

If the timestamper stops signing merkle roots, or publishes signed merkle roots without revealing the tree it represents, a user can only be confident that they are protected against a malicious reorg up to the earlier of the last update that was included in a signed merkle root, or the last update before the timestamper stopped revealing the content of their merkle trees. From that point on they are at risk of fraudulent reorgs, as they are now. If a user sees their timestamper exhibiting this type of dysfunction, they would be wise to switch to a different timestamper the next time they make an update. If the dysfunction began after the user sent their last update but before it was sent to the blockchain, they should switch to a new timestamper immediately to lock in their most recent update.

Note that since anyone can send a signed merkle root to the blockchain, there is no need for a user who wants immediate reorg protection to wait for someone else to do it; Once the timestamper has included their update in a signed merkle root, the user can do the blockchain timestamping themselves.

## Rationale and variations

### Why only guardrails?

We could put the whole system on a blockchain, but all available options have drawbacks.

Public blockchains, if sufficiently decentralized to make their usage meaningful, tend to have slow time to update, and can be unpredictably expensive due to scaling limitations. This proposal aims to maintain the current user experience of a user whose timestamper is well-behaved, which is difficult to guarantee with the mature public blockchains currently available.

Consortium blockchains consisting of trusted parties operating without crypto-economic incentives may be able to scale sufficiently and provide updates only somewhat slower than the current centralized directory. However such arrangements have rarely been successful in practice. One problem is that the consortium needs to consist of members with exactly the right amount of adversariality: If they are too well aligned with the people who ask them to participate they fail to provide an adequate check on the kind of behaviour they are supposed to prevent, and just run whatever update they are asked to by the person who would otherwise have been running the centralized server. This arrangement may even be worse than having a single trusted party, because responsibility is diluted and no single party can be blamed. If they are too adversarial, the consortium may fragment.

### Why sign messages individually instead of using the signature of the merkle roots?

Since the design calls for timestampers to put their messages in a signed tree, we could skip the individual signatures and instead rely on proofs against the signed merkle root. We propose individual signatures here only because it seems like a less disruptive change to the current functioning of the system (just add a "sig" field to the audit log instead of an entire proof) and secondly because the need to maintain the merkle tree in real time for all the accounts the timestamper has updated may have performance implications and complicate some strategies for scaling. These considerations aside, the design would work equally well without the individual signatures.

### Why not require the timestamper to publish signed merkle trees?

Assuming trees are not published synchronously with message signing (see previous question), adding a validity rule about the publication of an additional piece of data that you don't have yet adds complexity.

### Why sign `createdAt` and `CID` but not `nullified`?

Whether a message should be nullified can be verified independently by the client, and may be updated on receipt of new information about the DID history.

### Why not require the timestamper to publish blockchain updates?

The design makes blockchain timestamps entirely voluntary. Timestampers do not need to write to the blockchain at all, except when they update their signing key. We could instead require that updates be P2P timestamped. However this would stop timestampers processing updates they are unable to write to the blockchain for any reason, including a congestion spike resulting in high costs. By making blockchain updates voluntary and letting anybody do them, we allow people to publish only what is needed balancing the cost of making the updates at a particular time and the scale of the risk they are trying to address.

### Why use DID:ETHR for timestamper rotation keys?

Atproto supports two address standards, DID:Web and DID:PLC. DID:Web is unsuitable because we need to maintain a history of updates which cannot be tampered with even by the timestamper themselves. Using DID:PLC would add a degree of recursiveness to the system which makes it more confusing to analyse.

Timestamper key rotations are expected be unusual so the drawbacks of a public blockchain (cost per transaction, slow confirmation speed) should not be disqualifying. We propose DID:ETHR because it already exists as a well-specified standard. Using a simpler bespoke instead standard would also be viable.
