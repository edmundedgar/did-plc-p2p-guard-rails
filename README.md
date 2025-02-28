# P2P timestamping guardrails for DID:PLC

## tl;dr

The directory signs the updates it receives. Anyone can run a directory. You can switch directory with a DID update. A directory can sign merkle trees of messages it signed.

We use transactions on a public blockchain for: 

1. Forcing through updates that directory refuses to accept.
2. P2P-timestamping signed merkle roots for reorg-proofing. Anyone can do this whenever they like, and nobody has to.
3. Rotating the keys the directory signs with.

## Motivation

The centralized DID:PLC server model is working efficiently and economically. However, it breaks down if the server becomes malicious or broken.

This proposal attempts to mitigate or eliminate the problems that occur if it becomes malicious or broken, without compromising the functionality and low cost that users currently enjoy while it is running correctly.

## The current DID:PLC model

Conceptually the DID:PLC server performs two distinct functions:

### Timestamping

Users sign their own DID update operations, but the validity or invalidity of an operation can depend on what time it was sent in relation to other operations on the same DID. The DID:PLC server registers the time at which it received the operation and uses this to validate or invalidate each operation. Any other user who possessed the same updates that the DID:PLC server did, and also knew at what time they were received, could unambiguously reach the same conclusion about the current state of the DID.

This document will refer to the server performing this role as a "timestamper".

### Serving queries

DID:PLC server keeps a record of all updates, and serves the end user (or a mirror) either the current state of a DID or the record of updates to it (the audit log).

## Problems with the current DID:PLC model

The single source of authority about DID:PLC updates is the central DID:PLC server which serves updates over TLS. A malicious or broken DID:PLC could engage in:

### Censorship

The server can maliciously refuse to accept updates. 

It can also refuse to tell users and mirrors which updates it has accepted, in which case it will not be possible to run an honest mirror, so users cannot get information from any other source even if there is one they trust that was online when the updates were made.

This differs from the design elsewhere in atproto, which uses self-certifying data to ensure that although users may get their data from a semi-trusted server such as a relay or appview, they can always get the data from somewhere else and verify its accuracy if the server misbehaves.

### Malicious reorgs

The server can lie about the relative times at which it received messages.

       time t           time t+1           time t+2            time t+3

    Genesis[Key 1] --------------------> Update1[Key 2] -------------------------->
                   |                    (timestamped t2)
                   |
                   -------------------------------------> Update1b[Attacker key]-->
                                                         (falsely timestamped t1)

Exploiting the ability to lie about timestamps requires possession of a key that would normally only be held by the user or someone to whom they delegate. However, we can imagine two plausible cases where the attacker possesses such a key. In the first, the user rotates from an old rotation key to a new one, and does not secure the old key. In the second, the user initially creates an account with a third-party, for example bsky.social, but subsequently migrates their account elsewhere to avoid needing to trust the third-party. In both cases the user reasonably expects that their old key can no longer be used to manipulate their account, but this assumption is not guaranteed by the current design of the DID:PLC system.

### Poor unavailability

The server may simply start performing badly, in which case it would be better if clients could switch to an alternative.

## Our design

### Serving queries

We assume users will mostly continue to send their queries about DID state to a semi-trusted server, which may or may not be the same server that timestamps their updates. These servers will continue to serve updates for all users (but could be sharded). The rest of this document will refer to any party trying to get DID:PLC entries as a "client", but in practice it is likely that the "client" is a semi-trusted server, and the end user queries that.

### Signing messages

When a timestamper timestamps a message, it also signs the `CID` of the update and the `createdAt` field. Each entry in the audit log will then contain not only the user's signature on the update, but also the timestamper's signature on the record accepting the update.

Clients must consider updates invalid unless they have been either signed by the timestamper, or timestamped on the blockchain (see below).

### Managing timestamper keys

The timestamper must maintain a [did:ethr](https://github.com/decentralized-identity/ethr-did-resolver/blob/master/doc/did-method-spec.md) record. When it signs an update it must do so with a public key of type `EcdsaSecp256k1VerificationKey2019`, listed in their did:ethr record as a `verificationMethod` at the time specified in the `createdAt` field of the update.

### Changing timestampers

We add an additional field to the DID, `timestamperDid`. A user can update this field in their DID record in the same way they would update any other field in the DID record.

Entries with no `timestamperDid` specified are assumed to be using the DID representing the preexisting DID:PLC service.

### Forcing updates with P2P timestamping

If the timestamper is functioning correctly, a user can update their DID record with the timestamper as they do now. However, the user also has the option to send their update to the blockchain. In normal conditions this is slower and more expensive than sending it to their timestamper, so normally it would only be done if the timestamper is unavailable or refuses to accept their update.

Clients should read messages from the blockchain and include them as if they have been signed by the timestamper. This requires clients to have access to blockchain data, either by running their own node, by accessing a node run by someone else, or by receiving requests from a dedicated server they trust that sends them the relevant updates.

### Creating signed merkle trees

Timestampers should arrange snapshots of the updates they signed in a merkle tree and sign the root of the tree. At the timestamper's discretion this may be done immediately after every update, or only applied to periodic snapshots. They should share the content of this tree so that any user can verify the state of their DID record in any given tree.

Note that the failure to publish these signed merkle roots does not invalidate their updates, which have been individually signed.

### Timestamping signed merkle roots

Anyone may send a blockchain transaction to make a p2p timestamp of any signed merkle root at any time. Publishing the signed merkle root on the blockchain will allow a user in possession of the relevant part of the content of the tree to later prove that the timestamper had timestamped a given update at or before the time it was recorded on the blockchain. The blockchain timestamping of signed merkle roots is intended for reorg protection (see below) and no other purpose.

### Reorg protection

As we discussed earlier, a malicious or broken timestamper may sign messages that purport to have been received earlier than they really were. This can result in conflicting versions of the DID. Clients need to be able to choose between these conflicting versions.

We add the rule that that if a timestamper timestamps a version of the DID that conflicts with a version that has already been included in a signed merkle root timestamped on the blockchain, a client must honour the blockchain-timestamped version. (If multiple blockchain-timestamped versions conflict, they should honour the one with the earlier blockchain timestamp.)

Note that since both the creation and signing of the merkle root and its publication on the blockchain are optional, and the sharing of the data that would allow the construction of proofs against the merkle root is not enforced, a user can only be confident that their update is secure against a fraudulent reorg by the timestamper if the following conditions apply:

1. Their update is in a merkle root signed by the timestamper and published on the blockchain
2. They have the necessary intermediate tree node data to connect their update to that signed merkle root

In the scenario below, someone sends `Update1` to the blockchain for a P2P timestamp at time `t+3`. Clients will reject `Update1b` sent by the malicious timestamper at time `t+4` because they will see that it is in conflict with the version timestamped at time `t+3`. 

       time t         time t+1        time t+2          time t+3          time t+4

    Genesis[Key 1] --------------> Update1[Key 2] ------------------------------------------>
                   |              (timestamped t2)
                   |                               (blockchain timestamp)
                   |
                   -----------------------------------------------> Update1b[Attacker key]-->

                                                                   (falsely timestamped t1)

Had the malicious timestamper created `Update1b` between time `t+2` and time `t+3` sent it to the blockchain before `t+3`, their malicious update would have prevailed.

If the timestamper stops signing merkle roots, a user can only be confident that they are protected against a malicious reorg up to the earlier of the last update that was included in a signed merkle root. From that point on they are at risk of a fraudulent reorg. If a user sees their timestamper exhibiting this type of dysfunction, they would be wise to switch to a different timestamper the next time they make an update. If the dysfunction began after the user sent their last update but before it was sent to the blockchain, they should switch to a new timestamper immediately to lock in their most recent update.

Note that since anyone can send a signed merkle root to the blockchain, there is no need for a user who wants immediate reorg protection to wait for someone else to do it: Once the timestamper has included their update in a signed merkle root, the user can do the blockchain timestamping themselves if they so choose. This user's action will also protect all other users who made updates since the last blockchain update at no extra cost.

## Problems we don't solve

The following problems are mitigated by this design, but not completely solved:

### Immediate reorg protection

During the window between sending an update and somebody making a blockchain timestamp, a malicious timestamper can still maliciously reorg a DID using a key that was valid after the last timestamped update.

### Delayed reveal of account compromises

A malicious user in control of your rotation key can take control of your account. This is true regardless of how updates are timestamped. However, a consequence of the fact that we are only applying p2p timestamping to merkle tree roots without sending the updates they contain to the blockchain is that a malicious user in concert with a malicious timestamper can conceal the fact that they have taken control of your account until later. A user can know that there is something wrong with the timestamper, because they can see that it has published signed merkle roots without publishing the tree it represents, but they have no way of finding out whether their account has been compromised, and no way to protect their account in the event that it has.

## Rationale and variations

### Why sign messages individually instead of using the signature of the merkle roots?

Since the design calls for timestampers to put their messages in a signed tree, we could skip the individual signatures and instead rely on proofs against the signed merkle root. We propose individual signatures here out of practical concerns: Firstly because using individual signatures seems like a less disruptive change to the current functioning of the system (just add a "sig" field to the audit log instead of an entire proof) and secondly because the need to maintain the merkle tree in real time for all the accounts the timestamper has updated may have performance implications and complicate some strategies for scaling. These considerations aside, the design would work equally well without the individual signatures.

### Why not require the timestamper to publish signed merkle trees?

Assuming trees are not published synchronously with message signing (see previous question), adding a validity rule about the publication of an additional piece of data that you don't have yet adds complexity.

### Why sign `createdAt` and `CID` but not `nullified`?

Whether a message should be nullified can be verified independently by the client, and may be updated on receipt of new information about the DID history.

### Why not require the timestamper to publish blockchain updates?

The design makes blockchain timestamps entirely voluntary. Timestampers do not need to write to the blockchain at all, except when they update their signing key. We could instead require that updates be P2P timestamped as a condition for validity. However this would prevent timestampers from processing updates if they are unable to write to the blockchain for any reason, including a congestion spike resulting in high costs. By making blockchain updates voluntary and letting anybody do them, we allow people to publish only what is needed balancing the cost of making the updates at a particular time and the scale of the risk they are trying to address.

### Why use DID:ETHR for timestamper rotation keys?

Other parts of Atproto support two address standards, DID:Web and DID:PLC. DID:Web is unsuitable because we need to maintain a history of updates which cannot be tampered with even by the timestamper themselves. Using DID:PLC would add a degree of recursiveness to the system which makes it confusing to analyse.

Timestamper key rotations are expected be unusual so the drawbacks of a public blockchain (cost per transaction, slow confirmation speed) should not be disqualifying. We propose DID:ETHR because it already exists as a well-specified standard (albeit not a widely-adopted one). Using a simpler bespoke standard instead would also be viable. Another option would be to use DID:PLC, but with the additional restriction that the `timestamperDid` of a timestamper must be null so all updates to it would have to follow the "forced update" path.

### Would it make more sense to limit forced updates to timestamper selection?

Our design provides two paths for all DID updates, the standard timestamper path and the blockchain "forced update" path. An alternative would be to separate these and use the blockchain only to change your timestamper. This would have equivalent censorship-proofing properties, because you could always route around an uncooperative timestamper. Timestamper updates could be more compact, reducing gas usage. The downside is that if timestamper changes are still allowed via the non-blockchain route, clients now have to manage two different types of message which is more complex. Alternatively, if we stipulate that timestamper changes can only be done on the blockchain, users will have to spend gas doing something that could otherwise have been done for free by a cooperative timestamper.

### Why only guardrails?

We could solve the problems listed in "problems we don't solve" by removing the timestamping servers entirely and just putting all the updates on a public blockchain. However this will tend to be slower and more expensive than using a semi-trusted timestamping server, and this proposal aims to avoid damaging the experience of users to the extent that a server they trust is running honestly and working well.
