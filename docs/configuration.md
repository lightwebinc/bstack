# Configuration

What a member deployment needs, whichever member it is. The settings fall
into six groups, and each is owned by one role. The exact keys and flags
belong to each member and are in its own reference; the table at the end
points to them.

One rule runs through every group: **a setting that decides whom to trust,
or where objects and questions go, has no default.** A default header
source would send every verification question to a third party nobody
chose; a default host or submit endpoint would do the same with lookups
and objects. Members stop before opening a connection when one is missing,
rather than discovering it at the first object.

## At a glance

| Group | Publisher | Host | Reader | Default |
| --- | --- | --- | --- | --- |
| [Header source](#header-source) | to check a received payment | always | always | none |
| [Host set and quorum](#host-set-and-quorum) | no | no | always | the host a domain's manifest names; quorum 1 |
| [Wallet](#wallet) | always | no | no | the member's embedded wallet |
| [Topics](#topics) | always | always | through the lookup service name | the member's registered topic |
| [Submit endpoint and settlement leg](#submit-endpoint-and-settlement-leg) | always | no | no | none |
| [Lookup routes](#lookup-routes) | no | serves them | always | the base URL plus `/lookup` |

## Header source

Every proof is only as good as the headers it is checked against, so each
role names its own source and none takes headers from whoever answered.

| Role | Setting | Notes |
| --- | --- | --- |
| Reader | a header read API base URL (bfinger: `header_url`, `-header-url`) | bcommon's `headers` package reads a bridge's `/v1/root/{height}`; a 404 means the height is not held yet, and is kept distinct from an unreachable source |
| Host | the engine's chain tracker, pointed at a bridge's header read API | the bridge serves the headers it received on its own header lane |
| Bridge | an anchor for the initial chain and for gaps, and a minimum-work floor | [overlay-bridge configuration](https://github.com/lightwebinc/overlay-bridge/blob/main/docs/configuration.md): `-header-anchor` must serve `/v1/tip`, `/v1/root/{height}` and `/v1/header/{hash}`; `-header-min-bits` is `0x1d00ffff` on mainnet, and the default floor is for a lab only |

Point a reader at headers you or your organisation received: a bridge you
run, or one run by someone you already trust for that. Over plain unicast
the header source is the one hard dependency; nothing else in a deployment
needs the plane.

## Host set and quorum

A reader finds the host that serves a name from the name's domain, and the
replicas behind that host from DNS.

```text
   https://example.com/manifest.json          (BRC-180, https only, no fallback)
     metanet.overlays.ls_<name>  ─▶ https://overlay.example.com
                                         │
                            DNS A/AAAA   ├─▶ 192.0.2.30
                                         ├─▶ 192.0.2.31
                                         └─▶ 2001:db8::30
```

```json
{
  "metanet": {
    "handles": {
      "version": "1.0",
      "resolve": "https://example.com/.well-known/metanet-handles/resolve"
    },
    "overlays": {
      "tm_finger": "https://overlay.example.com",
      "ls_finger": "https://overlay.example.com"
    }
  }
}
```

| Setting | Notes |
| --- | --- |
| The manifest | one base URL per service; an absent entry means the domain does not offer it, and the reader must not go looking |
| Replicas | one A or AAAA record per host under the base URL's name, every host serving the same certificate name; add and remove replicas in DNS, never in the manifest or in client configuration |
| An explicit host (bfinger: `host`, `-host`) | for a domain that names no lookup service, or a read by bare identity key |
| Quorum (bfinger: `quorum`, `-quorum`) | how many distinct addresses must answer; two or more is what lets a reader see a fork between replicas |

bcommon's `hostset` resolves every address behind a name, keeps the name in
the URL, the Host header and the TLS handshake, and fans a request out
under a policy (first with failover, random, or all with a quorum). A 4xx
is an answer from a host that was up, not a reason to ask the next one.
[bfinger's host selection](https://github.com/lightwebinc/bfinger/blob/main/docs/host-selection.md)
sets out the production steps.

## Wallet

The publisher's wallet signs every script and input and funds every mined
transaction. Two shapes exist, both in bcommon:

| Shape | What it is | bfinger |
| --- | --- | --- |
| Embedded | the member's own key and coin in its state directory, shared with nothing else so two tools cannot double spend each other (`bwallet`) | `wallet = embedded`, the default |
| BRC-100 over the wallet wire | a wallet on the same machine that performs every derivation and signature; the wire carries no authentication, so a wallet that is not on loopback is refused (`wirewallet`) | `wallet = wire`, `wallet_url` (default `http://127.0.0.1:3301`) |

A wire wallet can also fund, sign and broadcast every mined transaction
through its own action flow (bfinger: `funding = wallet`). The member still
builds and signs the carrier, which no wallet may mine. A wallet that
broadcasts what it signs has no dry run, so bfinger requires `-yes` on every
owner command in that mode.

Wallet baskets are named per member and registered (below), so a wallet
holding coin for two members never mixes them.

## Topics

A topic is where a member's objects live on a host and on the plane. The
names follow bcommon's registry: `tm_<name>` for the topic manager and
`ls_<name>` for the lookup service, and a lab or test topic adds a suffix
(`tm_<name>_lab`) and is never used in production.

The same topic name appears in four places, and they must agree:

| Where | What it does |
| --- | --- |
| The publisher (bfinger: `topic`, default `tm_finger`) | names the topic in `x-topics` on every object-leg submission |
| The host | mounts the member's topic manager and lookup service on that name |
| The bridge (`-topics`) | the topics this host elected on the plane; a delivered object for a topic not listed is counted `unknown_topic`, the only signal that a subscription and a configuration have drifted apart |
| The domain's manifest | names the host that serves `ls_<name>` |

Registered names are frozen once anything has been committed on chain under
them. See [bcommon's registry](https://github.com/lightwebinc/bcommon/blob/main/docs/registry.md).

## Submit endpoint and settlement leg

A publisher has two legs, configured separately because they must never
share a connection.

| Leg | Setting (bfinger) | Values |
| --- | --- | --- |
| Object leg | `facade` | the base URL of a BRC-22 submit route: a bridge's facade, which admits locally and publishes once onto the plane, or, over plain unicast, a host's own submit route |
| Settlement leg | `settle` | `tcp:<host:port>` (bare Extended Format to an ingress, no acknowledgement), `rpc:<url>` (hex to a node), or `arcade:<url>` (an arcade installation, which reports a refusal before anything is published) |
| Proofs | `rpc`, `asset`, `proofs` | a node's RPC and asset API; `proofs = wait` publishes once the transaction is mined, `proofs = async` publishes when the network accepts it and collects the proof later, and needs a leg that answers |

On the host side, the bridge's facade is `-edge-ingress` (the delivery
slot's inner addresses, in failover order) and `-publish-source`; see the
[overlay-bridge configuration](https://github.com/lightwebinc/overlay-bridge/blob/main/docs/configuration.md).
A member that submits straight to the plane rather than through an engine
names the plane's ingress address and one topic.

## Lookup routes

A host serves each lookup service at its base URL plus `/lookup` (BRC-24),
with one question per POST.

- **Question classes.** A class is the set of member names in the question.
  A lookup service refuses a question carrying a member it does not define,
  rather than ignoring it, so no spelling can mint a priceable alias of a
  free question.
- **The free floor.** The base classes are priced zero on every conforming
  host. A host that charges for anything above the floor publishes its
  terms at its own route; a price is never carried in a record. Host-side
  charging is designed and not built in any member.
- **The reader asks the named host only.** bcommon's `lookup` client never
  runs discovery against public trackers: the host to ask is the one the
  domain named, or the one the reader configured.

## Running a host

A host that runs a member's module on the plane needs four things beside
the module itself:

| Setting | Why |
| --- | --- |
| Propagation off (no advertiser, or an empty tracker list) | the plane is the propagation; a host that also re-submits to its peers multiplies traffic, and the bridge's loop guard is sound only while the engine does not propagate |
| Chain tracker at a bridge's header read API | admission is checked against headers this host received |
| The topics mounted (the reference host: `OVERLAY_TOPICS`, `OVERLAY_MODULES`) | a module mounts only on topics the host names; a module that cannot be mounted stops the host before its port opens |
| A bridge in `feed` or `all` mode | `feed` delivers into the engine; `all` also serves the submit facade |

Over plain unicast the bridge is optional: the engine's own submit route
receives objects directly, and its chain tracker still needs a header
source the operator chose.

## Checking a deployment

| Check | What it shows |
| --- | --- |
| `bfinger doctor` | a bfinger home's local state, wallet, header source, node, facade and journal, contacting what is configured |
| The module's load line in the host log | which module, built with which bcommon version, is mounted on which topics |
| `overlay_bridge_tracker_roots_total{source="lane"}` against `{source="fallback"}` | whether the host's verification is fed by its own header lane rather than by the anchor |
| `unknown_topic` on the bridge's feed | a subscription and the bridge's `-topics` have drifted apart |

## Each member's own reference

| Member | Configuration reference |
| --- | --- |
| bfinger (repository, public) | [user guide, section 3](https://github.com/lightwebinc/bfinger/blob/main/docs/user-guide.md): the config file and its search order; [examples, configuration keys](https://github.com/lightwebinc/bfinger/blob/main/docs/examples.md): every key, its flag, default and who needs it; [host/README.md](https://github.com/lightwebinc/bfinger/blob/main/host/README.md): deploying the host modules, which have no configuration of their own |
| bcommon (repository, public) | [docs/registry.md](https://github.com/lightwebinc/bcommon/blob/main/docs/registry.md): every member's protocols, tags, magic, topics and baskets; [docs/versioning.md](https://github.com/lightwebinc/bcommon/blob/main/docs/versioning.md): pinning one tag in Go and TypeScript; [docs/dependencies.md](https://github.com/lightwebinc/bcommon/blob/main/docs/dependencies.md): the single dependency and its exact version |
| overlay-bridge | [docs/configuration.md](https://github.com/lightwebinc/overlay-bridge/blob/main/docs/configuration.md): every flag, the modes, and the startup checks that refuse a half-configured host |
