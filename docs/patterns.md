# The patterns

Each pattern below is stated with its intent, its mechanics, what it buys,
and its limit. They compose, and they are a shared language rather than a
mould: each member of the family combines them its own way and adds
patterns of its own where its problem demands one. The full argument, with
the economics and the standards each pattern rests on, is the pattern
paper at <https://1bsv.net/papers.html>;
[bfinger](https://github.com/lightwebinc/bfinger) is the first member to
demonstrate many of them live.

The foundations are ratified open BRC standards throughout: self-proving
objects (BRC-62/74/95/96), overlay state (BRC-22/24/88), key derivation
and wallets (BRC-42/43/100), discovery (BRC-169/180), payments (BRC-29),
keyed content (BRC-369), lookup markets (BRC-178), and the multicast
object plane (BRC-148/149). Nothing here required a new consensus rule or
a proprietary protocol; the design act is composition.

## 1. Mine the state, not the data

**Intent.** Use the chain for the one thing only a chain can do: put
events in an order nobody can rewrite. Keep everything else off it, where
storage is cheap, copies are free, and a retraction is real.

**Mechanics.** State lives in a chain of tiny mined outputs, the state
tokens, each spending the previous one and each committing to the current
record by a 32-byte hash. The record itself never appears in a block.

**What it buys.** Unforgeable ordering and history at a few hundred bytes
per update, with double-spend protection doing the work that version
vectors and consensus modules do elsewhere.

**Limit.** One chain per entity is a serialisation point, by design.
State that must change concurrently belongs to different entities with
different chains.

## 2. The carrier: a transaction built never to be mined

**Intent.** Give data a name, a proof, and a revocation lever, without
writing the data into a block.

**Mechanics.** The record rides an ordinary transaction whose fee is zero
and whose lock time is a century away, so it can never confirm. Its
transaction id is a 32-byte name for exactly those bytes. It spends an
output of a parent transaction that is mined, so it is provable by
ordinary SPV even though it will never be in a block.

**What it buys.** Content addressing, provability and retractability in
one device, using nothing but transaction rules every verifier already
implements.

**Limit.** A carrier is public to every host that carries its topic.
Confidential content is encrypted before it enters a carrier, never
after.

## 3. The funding tree: postage and admission in one

**Intent.** Make publication cheap for honest publishers and expensive
for floods, with no rate-limiter service to run.

**Mechanics.** One mined transaction creates M small outputs; each
published object spends exactly one. Every object is one proven parent
plus one Merkle proof: constant size, independently verifiable.

**What it buys.** Structural rate limiting (no unspent output, no
publication), constant verification cost per object, and a single point
of retraction for everything a tree funded.

**Limit.** The tree's width is chosen ahead of need; a publisher that
outruns it mines another tree.

## 4. The interval anchor

**Intent.** Give high-rate streams chain time and completeness proofs
without a miner fee per object.

**Mechanics.** Objects are numbered per producer and never mined. Once
per interval, one anchor transaction commits a Merkle root over the
interval's objects.

**What it buys.** An auditor proves the committed set for any interval
against block headers alone, and a missing object is a numbered hole that
any replica can fill. One fee per interval, regardless of rate.

**Limit.** Completeness is proven per interval, not instantly; the tail
of the current interval is evidence in flight.

## 5. Sub-store roots

**Intent.** Let one small record vouch for a large body of content,
revealed a member at a time or all at once.

**Mechanics.** A record commits to whole stores of further carriers by
Merkle root. A linked store publishes its member list; readers rebuild
the root and prove every member themselves.

**What it buys.** One mined output validates data amalgamated across many
stores, and the record stays small however large the content grows.

**Limit.** A root discloses the store's size, and an unlinked store is
unlisted rather than private; membership privacy needs an encoding that
carries a secret.

## 6. The witness

**Intent.** Make advancing the state require more than holding a key.

**Mechanics.** Each record publishes the hash of a secret and withholds
the secret. The next transition must reveal the preimage, which is
checked against the previous commitment.

**What it buys.** A second factor on every transition: a published record
proves its author held something it never showed anybody, and a stolen
key alone cannot advance the chain.

**Limit.** The withheld secret exists in exactly one place, the
publisher's own state, and is unrecoverable from the network by design.

## 7. The kill switch

**Intent.** Make retraction a mechanism instead of a request.

**Mechanics.** Spend the funding outputs the carriers depend on, and
every carrier hanging off them becomes a double spend at once. Conforming
hosts drop the records.

**What it buys.** One mined transaction retracts a record, or everything
a tree ever funded, with no takedown process and nobody to petition.

**Limit.** Retraction removes future availability from honest hosts. It
cannot un-copy what somebody already fetched, and nothing can.

## 8. Publish once

**Intent.** Delete the distribution tier.

**Mechanics.** A publisher submits an object once, named by topic; the
multicast object plane delivers it to every host subscribed to that
topic, complete and verbatim, proofs intact. Loss is detected and
repaired by the network itself.

**What it buys.** No fan-out servers, webhook farms, polling loops or
reconciliation jobs; audience size stops being a cost the publisher
engineers against. What the network moves is exactly what the reader
verifies.

**Limit.** Delivery at the same moment is an architecture property, not
physics; ordering across subscribers is never promised, and applications
that need total order put it in the data, with a chain. And the language
does not wait for the plane: every other pattern runs over plain
unicast, publisher to host, with a good block-header source as the only
hard dependency; the plane adds reliable distribution with global reach.

## 9. Hosts as replicas

**Intent.** Make host choice an availability question, never a trust
question.

**Mechanics.** Every subscribed host receives the same objects and admits
them on their own validity. There is no protocol between hosts and
nothing one host knows that another could not have received off the same
plane. Readers who care ask several hosts and require agreement.

**What it buys.** Interchangeable hosts, load balancing that is safe to
do simply, and a market where hosts compete on availability, latency,
retention and richer questions, never on custody of the data.

**Limit.** The cost of a bad host is a refusal and a retry; disagreement
between replicas is surfaced to the reader, who decides.

## 10. The free base answer

**Intent.** Keep participation free, and keep it anonymous.

**Mechanics.** A price attaches to a question class, never to part of an
answer. The base classes are free on every conforming host; a priced
class may not be a superset of a free one; a question carrying an unknown
member is refused, so no spelling manufactures a priceable alias of a
free question.

**What it buys.** A directory that can never become unreadable, and
anonymous reads at the floor: a host that charged for the base answer
would have to identify the payer. The floor is a floor, not a ceiling:
richer questions, history, exports, availability proofs and keyed
content are all open to pricing above it, and a paid read is
attributable by nature, which is exactly why the anonymous base answer
is pinned beneath it.

**Limit.** A restriction is a property of bytes, and only encryption
produces one; a service fee is a property of a socket. A priced lookup
sells availability and convenience, never exclusivity, and must never be
presented as access control. Paid content that must stay paid is keyed
content: the payment releases the key.

## 11. Money beside the data

**Intent.** Keep the delivery meter and the money separate, and leave
every payment topology open.

**Mechanics.** Any verified identity key is already a payable
destination. The simplest leg derives a fresh output from it, pays point
to point, and tells the recipient how to claim it. Richer shapes build
on the same substrate: a payment can travel inside a record and settle
when the recipient claims it; a bounty can be published as a record that
competing providers race to claim, singly or as a series funding ongoing
service; a payment can split across many receivers; and an interactive
payment flow can be settled by an overlay itself, its topic rules
judging the outcome at every host identically.

**What it buys.** No address reuse, a payment graph that is not a
broadcast artifact unless the design wants it to be, and revenue seams
at every edge that faces an audience: priced question classes,
subscriptions, bounties, markets across competing hosts. The delivery
meter never bills a payment as a payment.

**Limit.** A broadcast shows everyone the same bytes, so a payment that
must reach one party is derived per recipient, and a multi-party
settlement puts its judging rules in the topic, where every host applies
them alike. Notice delivery is a channel of its own; automating it is
the message-box member's job.

## 12. Sovereign verification

**Intent.** Never let the party who served the answer also supply the
yardstick.

**Mechanics.** Every object carries its Merkle proofs; every reader and
every host checks them against block headers it received itself, from a
header lane, never from whoever answered the lookup.

**What it buys.** The chain of "our API says so" becomes a chain of
arithmetic anyone can rerun, and a host that edits a record produces
something that fails the check rather than something believed.

**Limit.** SPV proves inclusion and ancestry; a reader with no host it
trusts still needs an index to prove a spend, and designs say so rather
than implying otherwise.
