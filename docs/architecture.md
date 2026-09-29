# Architecture

The reference architecture every member of the family shares: four roles,
one plane in two postures, and a small set of objects that carry their own
proof between them. Each member's repository documents its own instance of
this shape; where a member departs from it, the member says so in its own
specification. The patterns named here are stated in full in
[patterns.md](patterns.md).

Two conventions hold in the diagrams. `══▶` is delivery by the plane, and
`─▶` is an ordinary connection. Anything marked designed is not built, and
is named so that nobody plans around it.

## The whole shape

```text
  publisher                                          reader
  ─────────                                          ──────
  wallet ─▶ build and sign                           resolve name ─▶ identity key, base URL
    ├─ settlement leg ─▶ ingress, node or arcade ─▶ miner ─▶ chain
    └─ object leg ─▶ submit endpoint                          │
                          │                                   │ header lane
                          ▼                                   ▼
                  ╔════════════════╗                    header store ─▶ reader's header source
                  ║  object plane  ║                          │
                  ║  (or unicast)  ║                          │ chain tracker
                  ╚═══════╤════════╝                          │
             ╔════════════╬════════════╗                      │
             ▼            ▼            ▼                      │
          host A       host B       host C  ◀─────────────────┘
          engine, topic manager, lookup service
             │            │            │
             └────────────┴────────────┴─▶ lookup answers as BEEF ─▶ reader
```

The publisher never talks to a host's lookup route and the reader never
talks to the publisher. Everything between them travels as objects that
carry their own proof, which is why a host can be swapped, added or lost
without anyone reconfiguring anything.

## The four roles

| Role | Holds | Talks to | Never needs |
| --- | --- | --- | --- |
| Publisher | its keys, its coin, the secrets its records commit to | a settlement leg, a node for proofs, one submit endpoint | any host's lookup route, any reader |
| Plane | nothing of record | publishers at ingress, subscribed hosts at egress | to parse past an object's leading marker |
| Host | the admitted objects, indexed per topic | the plane (or publishers, over unicast), readers, its header source | another host |
| Reader | a header source it chose, and, where a member pins keys, its pin file | a domain's discovery documents, any host, its header source | the publisher |

The reader is stateless apart from its pin file. The host's storage is the
only state in the system that is not on the chain or in the publisher's
own files, and it is a replica: nothing one host knows could not have been
received by another off the same plane.

## The plane, in two postures

### Multicast: the BEEF object plane

The publisher submits an object once, named by topic, and the plane
delivers it to every subscribed host, complete and verbatim, proofs intact.
Loss is detected and repaired by the network. On the host's side a bridge
terminates delivery and hands the object to an unmodified engine through
the submit interface the engine already serves; the same bridge serves a
submit facade that admits locally and then publishes once onto the plane.

```text
   up                                            down
   publisher ─▶ facade (POST /submit)            plane ══▶ bridge feed ─▶ POST /submit ─▶ engine
                  ├─▶ local engine (real answer)       ══▶ bridge feed ─▶ POST /submit ─▶ engine
                  └─▶ one publication ══▶ plane        ══▶ ...
```

The engine's own propagation is switched off in this posture: the plane is
the propagation. Without it, every host that admits an object re-submits it
to every other, and between N hosts one object crosses the network on the
order of N squared times. The bridge is
[overlay-bridge](https://github.com/lightwebinc/overlay-bridge), and the
design is in _The overlay bridge_ (Papers, in the [README](../README.md)).

The plane carries three things a member uses:

| Plane piece | Standard | What a member uses it for |
| --- | --- | --- |
| Object lane | BRC-148, BRC-149 | every object a publisher submits, delivered to every host of the topic |
| Header lane | BRC-135 | bare block headers, checked for work by the bridge and served as a chain tracker |
| Submit facade | BRC-22 | the publisher's one submission, forwarded to a local engine and then published once |

A member that is not an overlay application at the host end takes delivery
the same way: bgateway's router reads BRC-149 delivery records straight
from a delivery stream, with no engine behind it.

### Unicast: peer to peer to each host

Every pattern runs without the plane. The publisher submits to a host's own
BRC-22 submit route over an ordinary connection, and the host admits the
object exactly as it would a delivered one. Reaching more hosts is then
more submissions, or the overlay's own propagation between hosts. The one
hard dependency in this posture is a good block-header source; the plane
adds reliable distribution with global reach, and nothing in the objects,
the host modules or the reader changes between the two.

## The overlay host

A host is a released overlay services engine with a member's module mounted
on it. The module is a topic manager (BRC-22), which decides what the host
admits, and a lookup service (BRC-24), which answers questions about what
was admitted.

```text
   delivered or submitted BEEF
            │
            ▼
   engine ─▶ topic manager ──── admits outputs on their own validity,
     │           │               retains the coins a submission spends
     │           ▼
     │      lookup service ──── indexes admitted outputs, answers question
     │                           classes, refuses a question with an
     │                           unknown member
     ▼
   chain tracker ─▶ header source this host chose (a bridge's header API)
```

The module never imports the engine. It is written against the engine's
declared interfaces, which bcommon's TypeScript package
(`@lightwebinc/bcommon`) reproduces structurally. A host loads it by path
and mounts its topic managers and lookup services under their names.
bfinger builds its module into a single file that imports nothing but
`@bsv/sdk`, so the module shares the host's copy of the SDK.

Three properties of every conforming host:

- **Admission is on validity, never on origin.** An object is admitted
  because its proofs check against the host's own headers and its topic
  rules pass, whoever submitted or delivered it.
- **A lookup service rebuilds itself.** The engine does not replay past
  admissions into a lookup service on start, so a module that keeps an
  index in memory offers a restore that the host calls before it serves.
- **The base question is free.** A price attaches to a question class,
  never to part of an answer, and the base classes are priced zero on
  every conforming host.

bfinger's modules, `tm_finger` and `ls_finger`, are the built example; see
its [host/README.md](https://github.com/lightwebinc/bfinger/blob/main/host/README.md).

## The committed record

The committed record is the shared starting shape for state that an overlay
keeps. It was first written down as
[bfinger's specification](https://github.com/lightwebinc/bfinger/blob/main/docs/committed-record.md);
members that use it cite it, and members that need a different shape write
their own.

| Object | Mined | On the plane | What it is |
| --- | --- | --- | --- |
| Record `S` | no | inside the carrier | the state, as canonical bytes the member's schema defines |
| Carrier `K` | never | yes, as BEEF | a transaction whose fee is zero and whose lock time is a century away, carrying `S` in its output |
| Commitment `C` | in the token | in the token | `C = txid(K)`, a 32-byte name for exactly those bytes |
| State token `O` | yes | yes, as BEEF | a tagged PushDrop output `[tag, C]`; each update spends the previous token |
| Funding tree `F` | yes | yes, as BEEF | one transaction with M small outputs; each carrier spends one |

```text
   funding tree F (mined)
     ├── out 0 ──▶ carrier K1 (never mined) ── carries S1
     ├── out 1 ──▶ carrier K2 (never mined) ── carries S2
     └── ...
                          ▲                          ▲
                          │ C1 = txid(K1)            │ C2 = txid(K2)
   token O1 (mined) ──spends──▶ token O2 (mined) ──spends──▶ ...
```

What each piece buys:

- **The token is the order.** Tokens form a chain per entity, each spending
  the last, so history cannot fork and double-spend protection does the
  work that version vectors do elsewhere. A few hundred bytes reach a block
  per update, and the record never does.
- **The carrier is the name, the proof and the lever.** Its transaction id
  names the bytes; it is provable by ordinary SPV through the mined funding
  output it spends; and spending that funding output some other way makes
  the carrier a double spend, which conforming hosts drop. That spend is the
  kill switch: one mined transaction retracts a record, or everything a tree
  funded.
- **The funding tree is postage.** No unspent output, no publication: rate
  limiting is structural, and every object is one proven parent plus one
  Merkle proof, constant in size.

A transition is one mined transaction and two plane objects, the carrier and
the token, both admitted by the same topic manager. A record may also carry
a witness commitment, the hash of a secret the next transition must reveal,
and roots over whole sub-stores of further carriers, so one small record
vouches for a large body of content.

Members vary the shape where their problem demands. bgateway keeps the
funding tree and the unmined carrier and drops the state token: each carrier
holds a batch of packets, and the router checks the tree once and one
signature per batch. The interval anchor, one mined root per interval over
many unmined objects, is the designed shape for high-rate streams.

## Publishing: two legs that never share a socket

```text
   publisher
     ├─ settlement leg: one mined transaction ─▶ tcp: bare EF to an ingress
     │                                        ─▶ rpc: hex to a node
     │                                        ─▶ arcade: an arcade installation
     │     └─ proof: waited for, or collected later and published again
     │
     └─ object leg: Atomic BEEF, one topic per POST ─▶ submit endpoint
```

The settlement leg carries a transaction to a miner; the object leg carries
a BEEF to a topic. The plane's ingress fixes a stream's grammar from its
first bytes, so a BEEF written down a settlement socket does not fail
loudly, it poisons the stream. bcommon's `publish` package offers no call
that accepts both. A carrier is never given to the settlement leg.

The proof for a mined transaction is either waited for before publishing
or collected afterwards; when it is collected later, the proven BEEF is
submitted once more and every host verifies the path against its own
headers. A leg that acknowledges nothing (the bare ingress) cannot be used
without waiting, because there the proof is the only evidence the
transaction was taken at all.

## Reading

```text
   reader
     ├─ discovery: https://example.com/manifest.json ─▶ resolve endpoint, lookup base URL
     │             (BRC-169, BRC-180; https only, no fallback)
     ├─ host set: every address behind the base URL ─▶ ask one, or N for a quorum
     ├─ lookup: POST /lookup, one question class ─▶ answer as BEEF (BRC-24)
     └─ verify: proofs against its own headers, then signatures, then the
                member's own rules (sequence, witness, validity window, pin)
```

A domain's manifest names one base URL per service. The replica set lives
at that name, in DNS, and the reader resolves every address behind it. A
reader may ask several hosts and compare, and disagreement is surfaced,
never hidden: bfinger refuses a name whose replicas disagree
(`REFUSED-FORK`) rather than choosing one.

Where a member binds a name to a key, the reader pins the key on first
contact and refuses a changed key unless a rotation signed by the pinned
key names the new one. Without the pin a host could swap the key it answers
with, and every signature check would faithfully verify the impostor.

## Header sources and sovereign verification

Every object carries its Merkle proofs. Every host and every reader checks
them against block headers it received itself, never against headers
supplied by whoever answered the lookup, because then the same party
supplied both the claim and the yardstick.

```text
   chain ─▶ header lane ══▶ bridge: work checked, chained, stored
                                 │
                                 │  GET /v1/tip
                                 │  GET /v1/root/{height}
                                 │  GET /v1/header/{hash}
                                 │
                                 ├─▶ engine chain tracker (every admission)
                                 └─▶ reader header source (every proof checked)
```

| Consumer | Where its headers come from |
| --- | --- |
| Overlay host | its chain tracker, pointed at a bridge's header read API |
| Bridge | its own header lane, checked against a minimum-work floor, anchored and re-anchored from a header service its operator trusts |
| Reader | a header source it names itself; bcommon's `headers` package is a chain tracker over a bridge's header read API |
| bgateway router | a file of heights and roots, or whole headers, that the router trusts |

Three rules hold everywhere:

- **No default.** A header source, a host and a submit endpoint have no
  default in any member, because a default would send verification
  questions, or objects, to a party nobody chose.
- **Not found is not an error.** A header source that does not yet hold a
  height answers 404, and that is distinct from an unreachable one;
  collapsing the two would make an outage look exactly like a forged
  proof.
- **SPV proves inclusion, not absence of a spend.** A reader with no host it
  trusts still needs an index to prove an output unspent, and each member
  says so rather than implying otherwise.

## The payment leg

The delivery meter never bills a payment, and a payment never rides the
object plane as a payment. Money moves beside the data.

| Shape | What moves | Where it exists today |
| --- | --- | --- |
| Point to point | a fresh output derived from a verified identity key (BRC-29), plus a notice the recipient claims it with | built in bfinger (`pay`, `receive`); notice delivery is by hand until a message box carries it |
| Payment channel | a funded 2-of-2 per leg, cumulative commitments the payee gates service on, settlement committing a Merkle root of the usage | bflow, proof of concept (loopback networking only) |
| Funded gate | a payee's funded-state feed that a service reads before it serves | bflow serves it; bgateway's router reads it in a `funded` policy rule |
| Priced question class | a price in front of a question other than the base one | designed; bfinger's lookup service already sorts questions into classes and refuses unknown members |
| Keyed content | ciphertext on the plane, a payment that releases the key (BRC-369) | designed |
| Payment in an envelope, bounty, multi-receiver split | a payment carried by a record and settled when claimed, raced or divided | designed (envelopes in bbox, bounties in bstore) |

What each party pays, in every member that publishes. Delivery is metered
on elected volume, flat in the number of participants, and each tier
monetises the audience it owns.

| Flow | Who pays | For what |
| --- | --- | --- |
| Settlement | the publisher | miner fees on each mined transaction; carriers are never mined and pay nothing |
| Object leg | nobody | a submission to one endpoint |
| Delivery | each receiving host | delivered bytes; the publisher does not pay for its audience |
| Base lookup | nobody, on any conforming host | reads are free and anonymous at the floor |
| A payment | the payer | the amount, plus one miner fee |

## Where bcommon and bflow fit

```text
   member (bfinger, ...)                 bflow (proof of concept)
     │ Go: pins an exact bcommon tag        │ payment channels, metering,
     │ TS: host module bundles              │ reconciliation, settlement
     │     @lightwebinc/bcommon             │
     ▼                                      ▼
   bcommon ─▶ go-sdk (one dependency)     funded-state feed ─▶ bgateway router
     └─ docs/registry.md: every member's       (HTTP/JSON, no Go import
        protocols, tags, magic, topics,         across the two)
        baskets
```

**bcommon** is the machinery members choose to share, public and
application-neutral: an application supplies its schema, derivation, tags
and wallet profile as parameters, and nothing in the library names one.
Its packages line up with the roles:

| Role | bcommon packages |
| --- | --- |
| Publisher | `cbor`, `commit`, `store`, `pushdrop`, `carrier`, `mint`, `funding`, `bwallet`, `wirewallet`, `publish`, `nodeapi` |
| Host module | `@lightwebinc/bcommon` (TypeScript): canonical CBOR, store refs, the reader's derivation, field signatures, the funding and carrier decodes, the engine interfaces a module satisfies |
| Reader | `resolve`, `hostset`, `lookup`, `headers`, `verify`, `guard`, `knownkeys` |

It has one direct dependency, go-sdk, at an exact version, and its tests
compare its output byte for byte with vectors from an independent
generator. It also holds the registry of the identifiers members put on
chain, so that no two collide. bfinger is built on it today; bgateway and
bflow build on go-sdk directly.

**bflow** is where the richer payment shapes are proven before a member
depends on them: a payer and a payee agree a contract, open a half-channel,
meter on both sides, reconcile on cumulatives (the lower value settles),
and settle on chain. It runs in two modes, bound (an operator side and a
verify-only agent that holds no key) and sovereign (two identical stations,
each paying for what it receives), over a synthetic chain on loopback. It
binds to members through narrow seams rather than imports: bgateway's
router reads its funded-state feed. Its repository names every commercial
seam it does not build.

## Standards in play

The foundations are ratified open BRC standards; nothing here needed a new
consensus rule or a proprietary protocol.

| Area | Standards |
| --- | --- |
| Self-proving objects | BRC-62, BRC-74, BRC-95, BRC-96 |
| Overlay state | BRC-22, BRC-24, BRC-88 |
| Keys and wallets | BRC-42, BRC-43, BRC-100 |
| Outputs and carriers | BRC-48 (PushDrop), BRC-60 (the non-final device) |
| Discovery | BRC-169, BRC-180 |
| Payments | BRC-29 |
| Keyed content | BRC-369 |
| Lookup markets | BRC-178 |
| The object plane | BRC-148, BRC-149, and BRC-135 for the header lane |

Each member's repository lists the standards it implements, where in the
code, and which are designed and not built.
