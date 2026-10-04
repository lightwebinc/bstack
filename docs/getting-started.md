# Getting started

Four tasks, in the order most people meet them: read a member's record end
to end, publish one, run a member's host module on an overlay host, and add
a new member to the family. bfinger is the member used throughout, because
it is the one that exists end to end; each step links to the member's own
documentation rather than repeating it.

The shapes behind each step are in [architecture.md](architecture.md), and
every setting named here is in [configuration.md](configuration.md).

## 1. Read a record end to end

A reader needs one thing configured: a header source it chose. Everything
else is found from the name.

```console
$ go build ./cmd/bfinger
$ ./bfinger -header-url https://headers.example.com alice@example.com
alice@example.com
  VERIFIED   signature, sequence 7 (update), proof at height 912430
  key        02a1b2…9f3c  (pinned 2026-04-11)
  plan       shipping 2.0
  status     available
```

What happened, in order:

```text
   1. resolve   https://example.com/manifest.json ─▶ resolve endpoint ─▶ identity key
                                                  ─▶ ls_finger base URL
   2. host set  every address behind the base URL (DNS)
   3. lookup    POST /lookup, the base question ─▶ token, carrier, previous carrier
   4. proofs    token and carrier checked against the header source above
   5. rules     record matches the commitment, signatures verify, pin matches,
                sequence advances, witness matches, validity window open
```

To see each step, and to go further:

| Do this | To see |
| --- | --- |
| `bfinger verify alice@example.com` | the same checks, with the trace printed step by step |
| `bfinger -quorum 2 alice@example.com` | two replicas asked and compared; a disagreement is refused as a fork |
| `bfinger -host https://overlay.example.com <identity key>` | a read with no domain in the loop |
| `bfinger -json alice@example.com` | every field of the verified answer, for a script |

Reads carry no account, credential or payment, and the base answer is free
on every conforming host. The
[bfinger user guide](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md)
covers installation, what `VERIFIED` means, key pinning and every refusal;
[docs/examples.md](https://github.com/lightwebinc/bfinger/blob/main/docs/examples.md)
shows every command with the output it really prints.

## 2. Publish a record

A publisher needs three endpoints and some coin: a submit endpoint for the
object leg, a settlement leg for the mined transactions, and a node for
proofs.

```
# ~/.bfinger/config
header_url = https://bridge.example.com
facade     = https://bridge.example.com
settle     = tcp:192.0.2.20:8000
rpc        = http://192.0.2.10:8332
asset      = http://192.0.2.10:8090
```

```console
$ bfinger init                        # identity key and an empty wallet
$ bfinger receive notice.json         # coin from a BRC-29 payment someone sent you
$ bfinger create alice@example.com -set status=available -set plan="shipping 2.0" -yes
$ bfinger status "in a meeting" -yes  # each change is one transition
$ bfinger doctor                      # what is configured, and what it answered
```

`create` mints and publishes a funding tree, then, like every transition,
settles one mined state token and submits two objects, the carrier and the token, to the
submit endpoint. Through a bridge's facade, that one submission reaches
every subscribed host. Without `-yes`, every command that can spend builds
its transactions, prints them and sends nothing.

For the name to resolve, the domain publishes two documents: its
`/manifest.json`, naming the host that serves `ls_finger`, and a resolve
endpoint that answers the identity key for a handle and echoes the handle
it was asked for. You do not need to run the domain to publish; a reader
can always read you by identity key instead.

| Next | Where |
| --- | --- |
| Not waiting a block per change (`settle = arcade:<url>`, `proofs = async`) | [user guide, section 3](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md) |
| Signing through a BRC-100 wallet on this machine (`wallet = wire`) | [user guide, section 3](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md) |
| The manifest and resolve documents, field by field | [user guide, section 9](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md) |
| Rotating a key, retiring, and the kill switch | [user guide](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md), [owner flow](https://github.com/lightwebinc/bfinger/blob/main/docs/owner-flow.md) |
| Paying someone you just looked up (`bfinger pay`) | [user guide, section 10](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md) |

`fund` fills the wallet by mining coinbase, so it works only on a chain you
control. On any other chain the wallet is filled by `receive` and by change
from your own transactions.

## 3. Run a host module on an overlay host

A member's host side is a module: a topic manager and a lookup service
written against the released overlay engine's interfaces. The module never
imports the engine, so it runs on any host built on that engine.

**Build the module.** bfinger's builds to one file, with bcommon's
TypeScript package inlined and `@bsv/sdk` left as a bare import:

```console
$ make host                           # writes host/bundle/index.js
```

**Place it inside the host's tree**, beside the host's `node_modules`. Node
resolves a bare import from the nearest `node_modules` above the module
file, so a module placed elsewhere either fails to import `@bsv/sdk` or
carries a second copy.

**Mount it.** A host that loads modules by path names the topics and the
module file:

```
OVERLAY_TOPICS=tm_anytx,tm_finger
OVERLAY_MODULES=/app/modules/bfinger/index.js
```

On a host with no module loader, the host's own startup code calls the
module's default export with a small host object (a logger and a metrics sink, the `ModuleHost` shape in
`@lightwebinc/bcommon`), and registers the topic managers and lookup
services it returns with the engine under their names. Before the host
serves, it calls each lookup service's `restore(outputs, storage)` with the
unspent outputs of that module's topics and the engine's storage, because
the engine does not replay past admissions into a lookup service on start.

**Set the posture.**

| Setting | Why |
| --- | --- |
| The engine's chain tracker at a header source this host chose | every admission is checked against it |
| Propagation off, with a bridge feeding the engine | on the plane, the plane is the propagation |
| Propagation as the operator runs it, and no bridge | over plain unicast |

**Confirm it loaded.** Each mount logs the module and the library version
built into it, and a module that cannot be mounted stops the host before its
port opens:

```
finger module built with bcommon {bcommon}
module loaded {path, topics, lookups}
module lookup restored from storage {path, lookup, outputs}
```

Deploy a new module version to hosts before publishers use anything it
introduces: a host running an older module refuses what it does not know.
The whole contract, including the kill and restart behavior, is in
[bfinger's host/README.md](https://github.com/lightwebinc/bfinger/blob/main/host/README.md);
the bridge side is in
[overlay-bridge](https://github.com/lightwebinc/overlay-bridge).

## 4. Add a new member

### Answer the design checklist first

Every member's design document carries these near the top, in writing,
before anything is built. A design that cannot answer the first is not
built.

1. **The nearest existing thing, and the delta.** Name the closest thing in
   the BRC corpus and the wider BSV ecosystem that already does this, and
   what the new member shows that it does not. Search before inventing.
2. **Who pays, who is paid, and for what unit,** for writes and reads
   separately.
3. **The payment leg,** or the sentence saying why there is none. Every
   verified identity key is already payable.
4. **The scaling shape:** where the state lives, what a replica is, what the
   serialization points are (a per-entity chain is a per-entity lock; never
   a global one), and what is stateless. No coordinator, no queue between
   hosts.
5. **What is frozen at first publish:** derivation protocols and key ids,
   tags, record magic, wire formats, manifest shapes. They are decisions,
   not implementation details.
6. **Every egress path.** The default is none; each exception is stated.
7. **Bytes on the plane per operation,** and what is referenced by outpoint
   instead of carried, since every object is delivered to every subscribed
   host.
8. **The standards relied on,** with the sections cited, and any measured
   result with the versions it was measured at.
9. **The levers deliberately not taken,** and what each would open.

Two constraints hold for every member: the base answer is free on every
conforming host, and nothing that decides trust or egress has a default.

### Register the member's identifiers

Before the member freezes its contract, it registers every identifier it
will put on chain, into wallets or onto hosts, in
[bcommon's docs/registry.md](https://github.com/lightwebinc/bcommon/blob/main/docs/registry.md),
in the same change that adds them to its code:

| Identifier | Rule |
| --- | --- |
| Protocol name | a BRC-43 protocol id of at least five characters; the security level is part of the id |
| Key ids | fixed strings recorded, or the shape of a per-object id described |
| Tag | a two-byte ASCII prefix owned by one member, plus one type byte it assigns |
| Record magic | the member's tag prefix, a letter and a version byte |
| Topic and lookup service | `tm_<name>` and `ls_<name>`; a lab topic adds a suffix and never reaches production |
| Baskets | start with the member's name |

A reviewer checks the new rows against every existing one. A collision is
not cosmetic: two members deriving under one protocol derive the same keys,
and two members sharing a tag are indistinguishable to a topic manager.

### Lay out the repository

bfinger's repository is the built example of the layout:

```
.
├── cmd/<member>/        # the command: reader and publisher
├── internal/            # the member's own packages: schema, rules, questions
├── host/                # the host module (TypeScript), built to one file
├── docs/                # user guide, specification, architecture, privacy, examples
├── testdata/            # golden vectors
├── Makefile
├── LICENSE
├── NOTICE
└── LICENSE-THIRD-PARTY  # where a dependency's licence requires its text to travel
```

- **Pin bcommon at an exact tag** in `go.mod`, and pack the TypeScript
  package from the same tag for the host module, so one version names both
  halves. Build a release binary with `GOWORK=off` and confirm the linked
  version with `go version -m`. See
  [bcommon's versioning](https://github.com/lightwebinc/bcommon/blob/main/docs/versioning.md).
- **Take the parameters, not the defaults.** bcommon names no schema,
  derivation, tag or wallet profile; the member supplies its own.
- **Write the specification** with its frozen list, and the user guide with
  its limits stated plainly. Where the committed record fits, cite
  [bfinger's specification](https://github.com/lightwebinc/bfinger/blob/main/docs/committed-record.md);
  where it does not, write the member's own shape.
- **Keep money out of the delivery path.** A payment is its own leg, run
  beside the data rather than inside it.

### Add it to the family

Add a row to the family table in the [README](../README.md) and a
paragraph to [family.md](family.md), naming the patterns the member
demonstrates and the members it leans on, and mark it designed until its
repository exists.
