# The family

A shared starting kit, worn differently. Most members build from the
same parts: committed records on the multicast object plane, carriers
from funding trees, roots binding sub-stores, mined state only where
order matters, hosts as replicas, free base answers, and per-entity
chains as the serialization points. There is no coordinator and no
global lock anywhere in the family, and the delivery meter never
carries the money: payment shapes run from point-to-point legs to
bounties, multi-receiver splits, and flows an overlay itself settles.
Each member keeps the parts that serve it, departs where its problem
demands, and says so in its own specification.

The patterns themselves are stated in [patterns.md](patterns.md).

| Working name | The pattern it demonstrates |
| --- | --- |
| [bfinger](https://github.com/lightwebinc/bfinger) | versioned self-published state: a name resolves to a key, the key signs the state, every host answers with proof |
| blogs | high-rate append: batches to many archives at once, per-interval completeness proofs, many producers with no coordinator |
| bbox | a message box replicated by the plane: envelopes with structural postage, payments traveling inside them |
| borg | organizations: membership as certificates with real revocation, group changes totally ordered by a chain, keys that rotate on removal |
| bsecret | ciphertext-only hosts: grants, versions and rotations as mined facts an auditor verifies against headers alone |

## The members

**bfinger** is the first member to ship: ask what `user@domain`
currently claims about itself and get an answer that proves itself, from
any host, with payments derivable from the identity that just proved
itself. Its repository carries the committed-record specification, the
first written down; members that use it cite it, and members that need
a different shape write their own.

**blogs** turns the shared kit toward volume. Producers append signed,
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

**borg** gives the family its organizations: membership as certificates
that can actually be revoked, every change to a group recorded on that
group's own chain so order is unforgeable, and group keys that rotate
when someone leaves rather than lingering. Secret grants and delegations
stand on it.

**bsecret** stores secrets so that the hosts holding them learn nothing:
content is keyed before it is published, hosts hold ciphertext they
cannot distinguish from noise, and grants, versions and rotations are
mined facts. An auditor verifies the trail against block headers with no
cooperation from the operator.


## How the members lean on each other

The order is dependency, and it compounds. blogs proves the high-rate
pattern the later members reuse. bbox gives releases and payment notices
a transport, and builds the paying client once. borg gives bsecret its
membership, ordering and key rotation.

bfinger is live, and it is the first, not the template. Each member
stands on its own: its own repository, published when it ships, carrying
its own specification, its own user guide, its own patterns where the
shared language does not fit, and its limits stated in plain terms.
