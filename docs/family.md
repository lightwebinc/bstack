# The family

One substrate, worn differently. Every application here is committed
records on the multicast object plane: carriers from funding trees, roots
binding sub-stores, mined state only where order matters, hosts as
replicas, free base answers, and per-entity chains as the only
serialisation points. There is no coordinator and no global lock anywhere
in the family, and money is bilateral even when the data is broadcast.

The patterns themselves are stated in [patterns.md](patterns.md).

| Working name | The pattern it demonstrates |
| --- | --- |
| [bfinger](https://github.com/lightwebinc/bfinger) | versioned self-published state: a name resolves to a key, the key signs the state, every host answers with proof |
| blogs | high-rate append: batches to many archives at once, per-interval completeness proofs, many producers with no coordinator |
| bbox | a message box replicated by the plane: envelopes with structural postage, payments travelling inside them |
| borg | organisations: membership as certificates with real revocation, group changes totally ordered by a chain, keys that rotate on removal |
| bsecret | ciphertext-only hosts: grants, versions and rotations as mined facts an auditor verifies against headers alone |
| bchat | a proof on every message: a company, its auditor and its partner hold the same bytes by construction |
| bstore | storage as a market: publish once, providers compete to hold, availability proofs as priced questions |
| bmedia | large sequenced objects to every edge at once, re-emitted onto a site's own multicast domain |
| bgateway | the carrier pattern applied to network traffic: batches as objects, routers as independently paid parties |

## The members

**bfinger** is the built, running member and the worked example for
everything else: ask what `user@domain` currently claims about itself and
get an answer that proves itself, from any host, with payments derivable
from the identity that just proved itself. Its repository carries the
committed-record specification the whole family shares.

**blogs** applies the same substrate to volume. Producers append signed,
numbered batches; every subscribed archive receives each batch at the
same moment; one anchor per interval commits the set, so any archive
proves completeness and any auditor proves time. A dropped batch is a
numbered hole filled from a sibling. Many producers, no coordinator.

**bbox** is the message box the rest of the family leans on: envelopes
addressed to a recipient, replicated by the plane instead of parked on
one server, retired by receipts, priced structurally by postage. A
payment can travel inside an envelope and settle when the recipient
claims it, which is what gives every other member its payment notices and
releases.

**borg** gives the family its organisations: membership as certificates
that can actually be revoked, every change to a group recorded on that
group's own chain so order is unforgeable, and group keys that rotate
when someone leaves rather than lingering. Chat rooms, secret grants and
delegations all stand on it.

**bsecret** stores secrets so that the hosts holding them learn nothing:
content is keyed before it is published, hosts hold ciphertext they
cannot distinguish from noise, and grants, versions and rotations are
mined facts. An auditor verifies the trail against block headers with no
cooperation from the operator.

**bchat** is team chat where the transcript defends itself: every message
carries a proof, every host of the workspace holds the same bytes, and a
company, its auditor and its partner can each run a host and agree by
construction. Private channels key their content per epoch; direct
messages ride bbox.

**bstore** turns storage into a market. A publisher puts an object on the
wire once; providers that subscribed to the bucket decide whether to
hold it, advertise their holding, and earn bounties, renewals and priced
availability proofs. Anyone with an interest in an object, not only its
publisher, can buy it more time.

**bmedia** applies the append pattern to media: each segment is a signed,
sequenced object delivered to every edge at once, served to players from
each edge or re-emitted onto a site's own multicast domain. Distribution
stacks downward: what arrives by broadcast can be re-broadcast.

**bgateway** is the research member: the carrier pattern applied to
network traffic itself. Batches of packets travel as objects, routers
verify provenance by SPV and are paid per flow, and a certified source
address costs the router nothing to check.

## How the members lean on each other

The order is dependency, and it compounds. blogs proves the high-rate
pattern the later members reuse. bbox gives releases and payment notices
a transport, and builds the paying client once. borg gives bchat and
bsecret their membership, ordering and key rotation. bstore gives
everything above it somewhere for content larger than a record. bmedia
composes the append pattern, the storage market and the keyed renditions.
bgateway stands apart, leaning only on the plane itself.

bfinger is live; the rest are designs on the same substrate, and each
member's repository is published when it ships, carrying its
specification, its user guide and its limits stated in plain terms.
