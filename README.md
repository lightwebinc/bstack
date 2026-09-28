# bstack

A pattern language, and the family of applications built from it.

Most of a modern application stack exists to do two jobs the application
never asked for: copying data to everyone who needs it, and getting
strangers to trust what arrives. bstack applications are built so that
neither job exists. State is carried by Bitcoin transactions that prove
themselves; a multicast network delivers each publication, once, to every
host that subscribed to it; hosts are interchangeable replicas competing
on service; and readers verify what they are handed instead of trusting
whoever handed it over.

```text
                       ┌───────────┐
                       │ publisher │
                       └─────┬─────┘
                             │  one flow, up
                             ▼
                  ┌─────────────────────┐
                  │   the object plane  │
                  │ (multicast delivery)│
                  └──┬───────┬───────┬──┘
                     │       │       │  every subscribed host,
                     ▼       ▼       ▼  at the same moment
                  ┌──────┐┌──────┐┌──────┐
                  │host A││host B││host C│
                  └──┬───┘└──┬───┘└──┬───┘
                     │       │       │  free lookups, any host
                     ▼       ▼       ▼
             readers, verifying against headers they received themselves
```

Four roles, and two of them never meet. The publisher signs state and
submits it once. The network delivers it. A host indexes what arrived and
answers questions. A reader asks any host, or several, and checks
everything against block headers it received itself. The publisher never
talks to a host, the reader never talks to the publisher, and no host
talks to another.

## The patterns in one breath

- **Mine the state, not the data.** A few hundred bytes reach the chain
  per update; the data rides transactions built never to be mined, so a
  retraction is a real retraction.
- **Publish once.** One submission reaches every subscribed host at the
  same moment, complete and verbatim; reliability is a network feature,
  not application code.
- **Hosts are replicas by transport.** No replication protocol, no
  coordinator, no origin server; adding a host is a subscription the
  publisher never learns about.
- **Verification is sovereign.** Every object carries its proofs and is
  checked against the reader's own headers; a wrong answer is caught by
  arithmetic, not reputation.
- **Users ride free.** Publishing costs miner fees and nothing else;
  reading is free and anonymous on every conforming host, by
  specification.
- **Receivers pay commodity rates.** Delivery is metered on elected
  volume, flat in the number of participants; each tier monetises the
  audience it owns.
- **Money is bilateral.** The network carries records, never payments; a
  payment derives a fresh destination from a verified identity key and
  travels point to point.

[docs/patterns.md](docs/patterns.md) states each pattern with its
mechanics and its limits. [docs/family.md](docs/family.md) maps the
applications to the patterns they demonstrate.

## The family

| Working name | One line |
| --- | --- |
| [bfinger](https://github.com/lightwebinc/bfinger) | ask what a name claims about itself, get an answer that proves itself |
| blogs | append-only streams to many archives at once, with per-interval completeness proofs |
| bbox | a message box replicated by the network, with payments travelling inside envelopes |
| borg | organisations: membership with real revocation, group changes in an unforgeable order, keys that rotate when someone leaves |
| bsecret | secrets whose hosts hold only ciphertext, with grants and rotations as facts an auditor verifies |
| bchat | team chat with a proof on every message |
| bstore | storage as a market: publish once, providers compete to hold |
| bmedia | large sequenced objects to every edge at once, re-emitted onward |
| bgateway | the carrier pattern applied to network traffic itself |

bfinger is built and running against a live network; the rest of the
family are designs that share its substrate, and their repositories are
published as each member ships. A common library carries the shared
machinery, so every member is a thin set of rules on the same foundation.

## Where the network fits

These are applications. They use a BSV multicast network (the BEEF object
plane, a bridge's submit facade and header lane, overlay hosts' lookup
routes) and are not part of it. The network side is documented in its own
papers:

- _The overlay object plane: publish BEEF once, and every overlay hears_:
  <https://1bsv.net/papers/overlay-object-plane.pdf>
- _The overlay bridge: an unmodified engine on the object plane_:
  <https://1bsv.net/papers/overlay-bridge.pdf>
- The pattern paper for this family is forthcoming at
  <https://1bsv.net/papers.html>

---

**Lightweb Inc.** · [lightweb.net](https://lightweb.net) ·
[1bsv.net](https://1bsv.net)

_Unbounded Solutions for a Small World™_
