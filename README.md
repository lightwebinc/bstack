# bstack

A pattern language for BSV overlay applications, and the family of applications
built from it.

Most of a modern application stack exists to do two jobs the application never
asked for: copying data to everyone who needs it, and getting strangers to trust
what arrives. bstack applications are built so that neither job exists. State is
carried by Bitcoin transactions that prove themselves; a multicast network
delivers each publication, once, to every host that subscribed to it; hosts are
interchangeable replicas competing on service; and readers verify what they are
handed instead of trusting whoever handed it over.

```text
                     ┌───────────┐
                     │ publisher │
                     └─────┬─────┘
                           │  one flow, up
                           ▼
        ┌────────────────────────────────────┐
        │          the object plane          │
        │        (multicast delivery)        │
        └────┬─────────────┬─────────────┬───┘
             │             │             │  every subscribed host,
             ▼             ▼             ▼  at the same moment
        ┌────────┐    ┌────────┐    ┌────────┐
        │ host A │    │ host B │    │ host C │
        └────┬───┘    └────┬───┘    └────┬───┘
             │             │             │  lookups, any host
             ▼             ▼             ▼
  readers, verifying against headers they received themselves
```

Four roles, and the pairs you would expect to be coupled never need to touch.
The **publisher** signs state and submits it once. The **network** delivers it.
A **host** indexes what arrived and answers questions. A **reader** asks any
host, or several, and checks everything against block headers it received
itself. Publishing never involves a host directly, reading never involves the
publisher, and no host talks to another. That separation is a property of
delivery, not a wall between the parties: a reader can always approach a
publisher directly, to arrange exclusive access to keyed content, for instance.
Nothing in delivery or verification ever depends on that contact happening.

## The patterns in brief

- **Mine the state, not the data:** a few hundred bytes reach the chain per
  update; the data rides transactions built never to be mined, so a retraction
  is real.
- **Publish once:** one submission reaches every subscribed host at the same
  moment, complete and verbatim; reliability is a network feature.
- **Unicast works today:** the same objects travel peer to peer to each host,
  with a good block-header source as the only hard dependency; multicast adds
  reliable, global distribution.
- **Hosts are replicas by transport:** no replication protocol, no coordinator,
  no origin server; adding a host is a subscription the publisher never learns
  about.
- **Verification is sovereign:** every object carries its proofs, checked
  against the reader's own headers; a wrong answer is caught by arithmetic, not
  reputation.
- **Users ride free, and paid content sits above the floor:** publishing costs
  miner fees only, the base answer is free and anonymous everywhere, and richer
  questions, history, proofs and keyed content are open to pricing.
- **Receivers pay commodity rates:** delivery is metered on elected volume, flat
  in the number of participants.
- **Money moves beside the data, in any shape the design earns:** the delivery
  meter never bills a payment; legs run from point to point to bounties,
  multi-receiver payments and flows an overlay's own rules settle.

[docs/patterns.md](docs/patterns.md) states all twelve patterns with their
mechanics and limits.

## The family

Each member stands on its own, with its own repository, specification and
patterns where its problem demands them; the patterns are a shared language, not
a mould. bfinger shipped first, and it is only that: the first. Repositories are
published as each member ships.

| Member                                            | One line                                                                                                                    | Status                                                   |
| ---------------------------------------------------| -----------------------------------------------------------------------------------------------------------------------------| ----------------------------------------------------------|
| [bfinger](https://github.com/lightwebinc/bfinger) | ask what a name claims about itself, get an answer that proves itself                                                       | repository (public); live on mainnet, with a public host |
| blogs                                             | append-only streams to many archives at once, with per-interval completeness proofs | built (repository private); [Pattern 005](https://1bsv.net/patterns/blogs.pdf) |
| bbox                                              | a message box replicated by the network, with payments traveling inside envelopes | built (repository private); [Pattern 002](https://1bsv.net/patterns/bbox.pdf) |
| borg                                              | organizations: membership with real revocation, group changes in an unforgeable order, keys that rotate when someone leaves | built (repository private); [Pattern 003](https://1bsv.net/patterns/borg.pdf) |
| bsecret                                           | secrets whose hosts hold only ciphertext, with grants and rotations as facts an auditor verifies | built (repository private); [Pattern 004](https://1bsv.net/patterns/bsecret.pdf) |
| bchat                                             | team chat with a proof on every message                                                                                     | designed                                                 |
| bstore                                            | storage as a market: publish once, providers compete to hold and serve                                                      | designed                                                 |
| bmedia                                            | large sequenced objects to every edge at once, re-emitted onward                                                            | designed                                                 |

One shared piece sits beside the members:

| Piece                                             | What it is                                                                                                                                                              | Status                                        |
| ---------------------------------------------------| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------| -----------------------------------------------|
| [bcommon](https://github.com/lightwebinc/bcommon) | the Go library, with a TypeScript package for host modules, that carries the machinery members choose to share, and the registry of every member's on-chain identifiers | repository (public); pre-1.0, tagged `v0.1.0` |

[docs/family.md](docs/family.md) maps each member to the patterns it
demonstrates and to the members it leans on.

## Where the network fits

These are applications. They use a BSV multicast network (the BEEF object plane,
a bridge's submit facade and header lane, overlay hosts' lookup routes) and are
not part of it. They do not require it to exist: every pattern runs over plain
unicast to the hosts, peer to peer, with a good block-header source as the only
hard dependency. The network is what makes distribution reliable and global.

## Documentation

- [Architecture](docs/architecture.md): the reference architecture every member
  shares: the four roles, the plane in both postures, the overlay host and its
  module, the committed record, header sources and sovereign verification, the
  payment leg, and where bcommon fits
- [Configuration](docs/configuration.md): what a member deployment needs,
  whichever member it is: header source, host set and quorum, wallet, topics,
  submit endpoints, lookup routes, and each member's own configuration reference
- [Getting started](docs/getting-started.md): read a record end to end, publish
  one, run a host module on an overlay host, and add a new member
- [Patterns](docs/patterns.md): the twelve patterns, each with intent,
  mechanics, what it buys and its limit
- [Family](docs/family.md): the members, the pattern each demonstrates, and how
  they lean on each other

## Papers

- _The overlay object plane: publish BEEF once, and every overlay hears_:
  <https://1bsv.net/papers/overlay-object-plane.pdf>
- _The overlay bridge: an unmodified engine on the object plane_:
  <https://1bsv.net/papers/overlay-bridge.pdf>
- _The bstack: publish once, prove everything, and let users ride free_:
  <https://1bsv.net/papers/bstack-patterns.pdf>

---

**Lightweb Inc.** · [lightweb.net](https://lightweb.net) ·
[1bsv.net](https://1bsv.net)

_Unbounded Solutions for a Small World™_

_© 2026 Lightweb Inc. All rights reserved. This architecture is in rapid
development; everything here is subject to change without notice._
