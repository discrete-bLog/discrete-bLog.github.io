---
layout: post
title: CoinPool, exploring generic payment pools for Fun and Privacy
authors: [Antoine Riard, Gleb Naumenko]
---

We think that a wide range of second-layer protocols
(LN, vaults, inheritance, etc) will be used by average Bitcoin users. We
are interested in finding and addressing the privacy issues coming from the
unique fingerprints these protocols bring.

More specifically, we are interested in answering the following questions:
1. How bad are privacy leaks from on-chain txn of second-layer protocols
and how much is leaked via protocol-specific metadatas (LN domain names,
watchtowers, ...)?
2. How to establish a list of Bitcoin fingerprints and their severity to
inform protocol designers and clarify threat models?
3. What kind of sophisticated heuristics spies may use in the future?
4. How to mitigate privacy leaks? Should each protocol adopt a common
toolbox (scriptless scripts, taproot, ...) in its own way or should we
design a confidential-layer to wrap around all of them?
5. How to make the solution usable (cheaper, easier to integrate, safer)
for a daily basis?

We suggest CoinPool: a generic payment pool [0] as a solution to those
problems. Although the design we propose is somewhat a scaling solution, we
won't focus on this aspect. This work is rather an exploration of *how a
pool construction could serve as a TLS for Bitcoin, enhancing both on-chain
and off-chain privacy*.

Please share your opinion on the [mailing list post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-June/017964.html).

### Motivation: cross-protocols privacy

It has always been a challenge to make the on-chain UTXO graph more
private. We all know the issues with cleartext amounts, the linkability of
inputs/outputs, and other metadatas. Combining with p2p-level spying
(transaction-to-IP mapping) or some other patterns leading to real-world
identities enable serious spying.

Protocols on top of Bitcoin (LN, vaults[1], complicated spending conditions
based on Miniscript, DLC [2] are even more vulnerable to spying because:
- each of them brings new unique fingerprint/metadata [3]
- known spying techniques against second-layer are currently limited to
trivial heuristics, but we can't assume spies will always this
unsophisticated

There is already a wiki list [4] attempting to cover all issues like that,
although maintaining it would be challenging considering privacy is a
moving target.

Let's consider this example: Alice is a well-known LN merchant with a node
tied to a domain name. She always directs the output of channel closing to
her vault address. If she has another vault address on-chain with the same
unique unlocking script (like a CSV timelock with a specific delta) this
can be leveraged to cluster her transactions. And since one of her
addresses is tied to a domain name, all her funds can now be linked to a
real-world identity.

In theory, one may use CoinJoin-like solutions to mask cross-protocol
on-chain transfers. Unfortunately, robust designs like CoinSwap depend on
timelocking coins, extensive use of the on-chain space, and paying fees to
provide sufficient privacy, as we explain further. These properties imply
we can't expect users to be using strong CoinSwaps by default.

That's why instead of specialized high-latency, high-chain-use
CoinJoin-style protocols, we propose CoinPool: a low-latency, generic
off-chain protocol used to be wrapped around any other protocol. CoinPool
is based on shared UTXO ownership. It may reasonably improve on-chain
privacy while avoiding latency and locked liquidity issues. CoinPool may
also reduce the on-chain use (thus, help to scale Bitcoin) if participants
cooperate sufficiently.

We do believe that CoinSwap and other CoinJoins are of interest, but we
have to consider the trade-offs and choose the best tool for a job to make
privacy usable with regards to user resources. We will compare CoinPool to
CoinSwap in more detail later in this write-up.

### Extra-motivation: on-chain scalability

Even though it's not the main focus of this proposal, we also want to
mention that since CoinPool is a payment pool, it helps with on-chain
scalability. More specifically:
1. Shared UTXO ownership allows to represent many outputs as one, reducing
the UTXO set in size.
2. The CoinPool design enables off-chain transfers within the pool, helping
to save the block space by committing fewer transactions on-chain.
3. CoinPool provides decent support for batching activities from different
users, also helping to have fewer individual transactions on-chain.

Since the CoinPool provides scalability benefits, users will be even
incentivized to join CoinPools due to the conservative chain resources
usage and such enjoy privacy as a side-effect.

### CoinPool design

A CoinPool must satisfy the following *non-interactive any-order
withdrawal* property: at any point in time and any possible sequence of
previous CoinPool events, a participant should be able to move their funds
from the CoinPool to any address the participant wants without cooperation
with other CoinPool members.

The state of a CoinPool is represented by one on-chain UTXO (a funding
multisig of all pool participants) and a set of transactions stored by the
participants along with signatures allowing to spend that UTXO. This UTXO
is a Taproot output, where the leaves in the Merkle tree represent pool
participants.

#### Transactions

A CoinPool UTXO can be spent by two types: Pool_Tx and Split_Tx.

A Pool_Tx enables cooperatives updates of the pool, e.g a participant
exiting the pool or off-chain internal transfers. This transaction is used
to spend the key branch of the Taproot tree of the CoinPool UTXO.
Signatures for a Pool_Tx should be exchanged "on-demand", the moment
parties decide to update the CoinPool state collaboratively, In practice,
this would happen upon a request of a pool participant.

A Split_Tx enables a unilateral exit from the CoinPool, in case it's not
possible to use a cooperative Pool_Tx path. This transaction spends the
UTXO via the Merkle branch into two outputs:
- a _withdraw_ output paying to the pool participant who initiated a
transaction
- a _recursive_ output paying to the new instance of a CoinPool, which
contains all the same participants except the one who just withdrew

The design of the unilateral Split_Tx depends on what can be achieved with
Bitcoin Script. The main challenge is to enforce the second output of the
Split_tx s that the participant who exists can't take all the funds
unilaterally. We will talk more about the updates to Bitcoin Script which
would allow more advanced pools later (Scaling section).

For now, we will *focus on the Script capabilities of today*, per which
spending a Split_Tx requires signatures from all pool participants. Since
Split_Tx is a unilateral exit, parties are required to exchange signatures
for *any possible state of the pool* in advance, to handle the *any-order
withdraw* requirement. The exchange should happen when a pool is created.

#### Operations

There are three types of operations against a CoinPool: create, update,
withdraw.

Per *creation*, participants agree on a pool policy and commit inputs to a
funding transaction by sending a corresponding signature, created in a
secure "atomic" way (so that their funds can't be taken if other pool
participants are unresponsive). Participants also exchange their signatures
which would allow any participant to exit at any given time via a Split_Tx.

Per *update*, participants agree on a new coin distribution within the pool
tree. They can aggregate and split leaves of the tree, or rotate a target
output of a given leaf. E.g, a participant may choose to redirect coins to
a new pool right from the old pool and ask all the parties to agree on this
update. The previous state should then be revoked either via sequence
number (Eltoo) or adding the latest state as a child transaction from any
previous Pool_tx.

Per *withdraw*, a participant may submit either a Pool_Tx (after asking all
the parties for their signatures) or a Split_Tx (unilaterally). After that,
a new UTXO of the CoinPool would consist of all the remaining participants.

As an optimization, updates and withdrawals may aggregate changes to
multiple leaves within one transaction. A CoinPool may also optionality
allow new participants to *join* a pool on-the-fly, although trade-offs
should be considered.

#### Transaction Tree illustrated

We illustrate a CoinPool transaction tree with 3 leaves below. We use an
obvious optimization: if there are only 2 leaves left, the last transaction
doesn't have to commit to a new tree [5].

                                              Funding_tx
                                                 |
                                            [Taproot T]
                                                 |
                        _________________________|______________________________
                       |                                                        |
                       |                                                        |
                       |                                                        |
                   [leaf_A]                                                     |
                       ^                                                        |
                       |                                  ______________________|____________________
                    Split_TxA                             |                                           |
                    /      \                             |                                           |
                   /        \                         [leaf_B]                                    [leaf_C]
          [withdraw_A]   [Taproot T']                    ^                                           ^
                             ^                           |                                           |
                             |                        Split_TxB                                    Split_TxC
                          Pool_Tx                     /     \                                     /     \
                           /    \                    /       \                                   /       \
                          /      \           [withdraw_B]  [Taproot T'']                [withdraw_C]   [Taproot''']
               [withdraw_B]  [withdraw_C]                      ^                                          ^
                                                               |                                          |
                                                            Pool_Tx                                    Pool_Tx
                                                             /   \                                      /   \
                                                            /     \                                    /     \  
                                                  [withdraw_A]  [withdraw_C]                 [withdraw_A]   [withdraw_B]

### Scaling the Pool and the Any-Order problem

A conservative CoinPool indeed does not scale well: it requires generating
pruned Merkle Tree encumbering the _recursive_ output for any combination
of withdrawals at pool creation. For a tree of Alice, Bob and Carol, they'd
have to build (A,B,C), (A,C,B), (B,A,C), (B,C,A), (C,A,B), (C,B,A) trees.
Since the complexity is quasi-factorial, the conservative CoinPool design
is impractical for more than 10 leaves.

Instead of operating over every possible alternative statically (via
pre-signing *every* combination), the protocol may rely on the script
interpreter to do it dynamically, only enforcing an effective path among
alternatives.

A new primitive to enable this behavior can be implemented as an
accumulator, i.e a space-efficient cryptographic set representation
supporting testing for inclusion and element deletion.

Implementation of this delete-only accumulator can be done by introducing
or combining already-proposed primitives like a new sighash flag, using a
Taproot tree as an accumulator, a committed bitset with templated
operations, etc, ... The exact design is left for future research.

This primitive would enable re-committing the updated tree on the
_recursive_ output, that way preserving balances of other participants,
independently of order of withdrawals.

Such _recursive_ output needs to be spendable by remaining Split_txn. These
transactions are pre-signed and their inputs commit to the parent txid. To
alleviate this issue, Split_tx should be signed through SIGHASH_NOINPUT,
therefore enabling _recursive_ output. The spending Tapscript must be
present among the set of Taproot tree leaves.

### Intra-pool communication and pool policies

The CoinPool design assumes participants have to communicate regularly to
exchange transaction templates and signatures. It happens almost every time
a state of the pools is modified: pool creation, pool update, and
cooperative withdrawal. Selecting a communication channel (a mixnet,
centralized servers, publication boards, ...) should be done considering
the threat model, the cost, the expected latency.

Every instance of a CoinPool may be public (available for new participants
to join at any time), private (available to join via some out-of-band
communication), or something in the middle (based on anti-Sybil measures we
will discuss later).

### Protocol rebinding

Since the conservative CoinPool design every unilateral exit via Split_Tx
should be signed by all the parties in advance, every participant joining
the pool should define in advance which address the coins will be directed
to in a unilateral case.

However, participants may want to use their CoinPool funds to move their
pool funds to a new scriptPubkey (for example, to open a new LN channel).
To avoid using an intermediate single-address on-chain transaction for
these cases, participants should be able to rotate the Split_Tx output.

To avoid asking other participants of the CoinPool to sign a new update, a
multisig signature covering the Split_Tx and enforcing covenant semantic
may be signed with SIGHASH_SINGLE. That way, at any-time Alice can finalize
her Split_Tx by adding a new output and signing her with SIGHASH_ALL.

Since unilateral withdrawal from CoinPool is time-locked, integration
time-sensitive off-chain protocols (e.g, LN or DLC) must be done with extra
care.

Lastly, the limitations of the current mempool design should be taken into
account while using CoinPools, so that issues like mempool  pinning [6] are
not critical.

### Security/Privacy model

Similarly to CoinJoins, CoinPool provides privacy by breaking payment
sender/receiver linkability for an on-chain observer.
Common-input-ownership, address reuse, change address heuristics can't be
leveraged. A spy is forced to commit/lock funds to the pool, and
potentially overcome ant-Sybil measures. Internal CoinPool transfers also
remain private for an external observer.

The exact on-chain privacy efficiency of a given CoinPool depends on two
factors: intra-pool activities and exit activities. If the parties are
cooperative, intra-pool transfers never leave a footprint on-chain. Exit
activities always hit the chain, but if output rebinding is available, the
funds can be sent right to the target receiver outside the pool (e.g, cold
storage or even another pool), making on-chain analysis much more difficult
than what happens with regular transactions today.

Since it's possible for an attacker to join a pool, we have to consider
extra Sybil-resistance (beyond just depositing coins). Extra
Sybil-resistance may include a lock on withdrawal. This lock should not
limit intra-pool updates, so that honest users are not limited.
Additionally, the solutions suggested for CoinJoins may be used (fidelity
bonds, PoDle, etc).

### User Requirements

CoinPool introduces two requirements on users: one for security, one for
pool performance.

It requires persistent storage from the user. Since a unilateral withdrawal
assumes transmitting signatures received from other participants
beforehand, these signatures, corresponding Taproot output and Merkle
branches should not be lost or corrupted, otherwise a participant won't be
able to exit the pool without cooperation.

It also requires hot access to the signing keys, i.e being online to sign
updates which introduces a higher security risk.

These requirements are similar to the requirements for LN and vault
constructions, so we believe that the burden on a CoinPool participant is
reasonable as long as we consider second-layer protocols practical.

### Comparing to CoinSwap

A CoinSwap was recently proposed as a next step for on-chain bitcoin
privacy [6]. We will compare CoinPool to  CoinSwap in terms of high-level
properties, because there are no deployed CoinSwaps yet.

CoinSwap enables privacy-enhanced (in terms of on-chain footprint)
transactions, executed as a non-custodial atomic "trade" between two
parties willing to send someone else (not each other) coins at the same
time.

A CoinSwap between two parties cost at least two on-chain transactions.
However, since this minimal design leaks privacy due to the amount
correlation, more advanced CoinSwap constructions should be used, and they
would be even more costly.

The privacy-efficiency of CoinSwaps is thus defined by fees and time-value
parameters. For these reasons, the lower security requirements are likely
to be most-widely picked-up by thrift users. Participation in a CoinPool,
however, costs of a funding transaction fee (shared across all
participants), and a cost of the withdrawal (either unilateral or
cooperative). Off-chain updates within a pool are free.

With regards to linkability, CoinSwap breaks the UTXO ownership graph,
CoinPool has similar properties the moment the coins are withdrawn, but the
off-chain events inside the pool are likely to obfuscate it even further.
Every participant can make direct payments during pool lifetime, breaking
mapping between committed inputs and withdrawn outputs.

CoinPool output will be part of the broader taproot user set, therefore any
single-owned output may be confused as a pool one, hindering further
on-chain analysis even for non-CoinPool users.

With regards to linkability, CoinSwap completely breaks the link between
inputs and outputs, while CoinPool just largely obfuscates the link
(similarly to CoinJoins). CoinPool is capable of breaking the link for
those payments happening off-chain (e.g, simple transfers within the pool).

With regards to requirements, beyond requiring online keys, practical
CoinPools require new base layers primitives, namely Taproot,
SIGHASH_NOINPUT and delete-only accumulators. CoinSwap is deployable today,
although the client software should be built.

With regards to malicious participants, CoinPool provides some privacy (in
terms of on-chain analysis) if at least one other participant is honest,
which is the same assumption as CoinSwap. More specifying spying cost
analysis can be made when comparing particular configurations of a CoinSwap
and a CoinPools. We invite the community to develop a better model of
privacy adversaries, their resources and leverages to refine
privacy-enhancing protocols comparisons.

Although we claim that different properties of CoinSwaps and CoinPools make
them better for different goals, they can benefit from each other: e.g,
both of them may rely on Sybil-resistance mechanisms or federated message
boards for cooperation.

### Conclusion

We propose CoinPool: a payment pool construction to improve privacy against
on-chain data analysis. More specifically, it helps to hide the unique
footprint associated with the use of second-layer protocols.

We attempted to design CoinPools to make them usable for daily activities,
as opposed to specialized CoinJoin-style solutions. They usually don't
require paying fees and don't use the on-chain space *per-activity*. If
they are widely used, they can also help with on-chain scalability,
although we don't cover this aspect in detail.

CoinPool is a UTXO representing a Taproot tree, in which the leaves
represent the spending conditions for coins in the pool. We designed
CoinPool around the *non-interactive any-order withdrawal* requirement.

In the long-term, CoinPool appears as a good candidate for a scalable,
used-by-default privacy-enhancing technology. We emphasize there are
several challenges deploying CoinPools, the biggest one being scalability.
Making them practical requires introducing new on-chain primitives.

We are grateful to the wider privacy-community and on which of their work this
depends heavily. We also would like to thank the reviewers of this post.

#### References

[0] AFAIK, payment pools have been suggested by Greg Maxwell, although I
couldn't find any written evidence. The only description I know have been
made to me by Dave Harding during a BitDevs meetup. Yes I'm old enough to
have known when meatspace Bitcoin meetup was a thing.
<https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-April/017793.html>

[1]
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-April/017793.html

[2] https://github.com/discreetlogcontracts/dlcspecs/

[3] On protocol usage leak at the on-chain level see
https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-February/017633.html,
for an illustration see
https://b10c.me/mempool-observations/1-locktime-stairs/

[4] https://en.bitcoin.it/Privacy

[5] . A gist backup if it doesn't survive formatting :
https://gist.github.com/ariard/ab1e4c3a85e4816be21ee0e0f925e86b A common
notation to describe transactions tree and their scripts and ease reviewers
to verify their correctness would be great. Otherwise it's hard to gauge
the correctness of this kind of new protocol.
