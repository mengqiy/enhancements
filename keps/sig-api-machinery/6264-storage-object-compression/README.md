# KEP-6264: Storage Object Compression

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Measured on in-tree fixtures](#measured-on-in-tree-fixtures)
  - [Measured on production data](#measured-on-production-data)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: an encrypted cluster with no other lever](#story-1-an-encrypted-cluster-with-no-other-lever)
    - [Story 2: an unencrypted cluster](#story-2-an-unencrypted-cluster)
    - [Story 3: an operator who wants compression but not on Secrets](#story-3-an-operator-who-wants-compression-but-not-on-secrets)
    - [Story 4: an SRE during a rolling upgrade](#story-4-an-sre-during-a-rolling-upgrade)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [The at-rest byte format](#the-at-rest-byte-format)
  - [Why the ordering is not configurable anywhere](#why-the-ordering-is-not-configurable-anywhere)
  - [The write path](#the-write-path)
  - [Staleness, and why framing must not depend on the compression outcome](#staleness-and-why-framing-must-not-depend-on-the-compression-outcome)
  - [The read path](#the-read-path)
  - [Bounding decompression](#bounding-decompression)
  - [Integrity: what authenticates a frame](#integrity-what-authenticates-a-frame)
  - [API Priority and Fairness, and size accounting](#api-priority-and-fairness-and-size-accounting)
  - [Error classification and version skew](#error-classification-and-version-skew)
  - [Observability](#observability)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [Deprecation](#deprecation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
  - [Open Questions](#open-questions)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Appendix: measuring the compression level trade-off](#appendix-measuring-the-compression-level-trade-off)
- [Appendix: measuring compressor and decompressor memory footprint](#appendix-measuring-compressor-and-decompressor-memory-footprint)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Every write to etcd passes the serialized object through a `value.Transformer` chain that encrypts
it. Ciphertext is incompressible, so in an encrypted cluster no layer below kube-apiserver — not
etcd, not bbolt, not the gRPC transport — can reduce the storage footprint of an API object.

This KEP adds an opt-in compression stage that runs **before** the encryption chain on write and
after it on read, implemented as a single `value.Transformer` decorator. Because a decorator's
`TransformToStorage` compresses and then delegates, the effective ordering is
compress → encrypt → prefix, and the `k8s:enc:` envelope stays byte-for-byte outermost on disk.

Backward compatibility is a property of the bytes rather than of configuration. A compression frame
begins with `0x00`, which no kube-apiserver storage serializer can emit — protobuf begins
`6b 38 73 00`, JSON `{`, CBOR `d9 d9 f7`. So "is this value compressed?" is decided by one byte
comparison, with no flag, no probe, and no per-resource state. Consequently **decompression is
unconditional and cannot be disabled**; only the write path is gated. Disabling the feature, or
removing a resource from the policy, can never strand data.

Compression is opt-in per resource and off by default at every stage, including GA.

## Motivation

Large clusters accumulate etcd bytes that no existing mechanism can reclaim. For clusters using
encryption at rest the situation is absolute: etcd receives ciphertext, so etcd-side or bbolt-side
compression delivers exactly zero. Compression has to happen inside kube-apiserver, above the
transformer chain, or it does not happen at all.

Compression at this layer buys two things:

1. **etcd database size.** Fewer bytes per stored revision. The database file holds live revisions
   plus superseded ones retained until compaction, so the reduction applies to every revision etcd is
   holding.
2. **Backup, snapshot and raft replication volume.** The same reduced bytes are what
   `etcdctl snapshot save` writes and what raft replicates to peers.

It does nothing for watch-cache or apiserver heap usage, which hold decoded objects rather than
stored bytes, and nothing for small high-entropy objects — which is why a minimum size threshold
exists.

### Measured on in-tree fixtures

The round-trip fixtures in `staging/src/k8s.io/api/testdata/HEAD`, raw DEFLATE level 1:

| Fixture | protobuf | compressed PB | ratio | JSON | compressed JSON | ratio |
|---|---|---|---|---|---|---|
| `core.v1.Pod` | 13 572 | 2 773 | **4.89×** | 61 113 | 7 746 | 7.89× |
| `apps.v1.StatefulSet` | 13 077 | 2 693 | 4.86× | 64 780 | 8 501 | 7.62× |
| `apps.v1.Deployment` | 11 937 | 2 506 | 4.76× | 60 639 | 7 630 | 7.95× |
| `batch.v1.Job` | 12 241 | 2 608 | 4.69× | 62 072 | 8 033 | 7.73× |
| `core.v1.Node` | 1 363 | 653 | 2.09× | 4 521 | 1 242 | 3.64× |
| `core.v1.Endpoints` | 728 | 343 | 2.12× | 2 319 | 603 | 3.85× |
| `events.k8s.io.v1.Event` | 778 | 413 | 1.88× | 2 212 | 648 | 3.41× |
| `core.v1.Service` | 945 | 556 | 1.70× | 2 977 | 935 | 3.18× |
| `rbac.authorization.k8s.io.v1.Role` | 492 | 327 | 1.50× | 1 488 | 506 | 2.94× |
| `core.v1.Secret` | 441 | 299 | 1.47× | 1 293 | 460 | 2.81× |
| `core.v1.ConfigMap` | 427 | 298 | 1.43× | 1 267 | 462 | 2.74× |
| `coordination.k8s.io.v1.Lease` | 485 | 342 | 1.42× | 1 464 | 537 | 2.73× |

Two things this set actually tells us, neither of which is "the ratio is 4.9×":

**Most of these objects would never be compressed.** Seven of the twelve have a protobuf form under
1 KiB and therefore fall below the minimum size threshold — Secret, ConfigMap, Lease, Role, Event,
Endpoints, Service. Their 1.4–2.1× ratios are moot, and that is by design: those are
exactly the objects where compression costs CPU on every read and write for no durable benefit.

**The fixtures are optimistic, and now we can say by how much.** Every field is populated, which is
unrepresentative in both directions — a fixture Secret is smaller than a real one holding actual
data, while a fixture Pod is denser than a real one. The useful calibration is against real objects
of comparable size: at ~13.5 KB the fixture Pod claims **4.89×**, while real production Pods in the
~14.6 KB size class deliver **2.75×**. So for this resource the synthetic fixtures overstate by
roughly 1.8×. That is the main reason the fixtures are shown as a control rather than as evidence.

### Measured on production data

672 Pods sampled from `RequestResponse` audit events on a large production cluster, decoded into
typed objects, re-encoded to protobuf with `resourceVersion` cleared to model the storage layer, and
compressed at DEFLATE level 1 — the shipped configuration.

| Size class (protobuf p50) | Pods | stored p50 | ratio p50 | ratio p10 | ratio p90 |
|---|---|---|---|---|---|
| 3.5 KB | 72 | 1 986 | **1.75×** | 1.75× | 1.75× |
| 6.7 KB | 100 | 2 745 | **2.43×** | 2.41× | 2.45× |
| 14.6 KB | 100 | 5 308 | **2.75×** | 2.71× | 2.75× |
| 34.2 KB | 100 | 13 406 | **2.56×** | 2.55× | 2.71× |
| 76.6 KB | 100 | 18 669 | **4.10×** | 3.79× | 4.25× |
| 162.5 KB | 100 | 51 650 | **3.36×** | 2.35× | 4.98× |
| 278.5 KB | 100 | 45 211 | **6.15×** | 6.01× | 6.18× |

**These size classes must be read independently and must not be pooled.** Each was sampled as up to the
100 largest Pods below a size threshold — the smallest class yielded only 72 — so the sample says nothing about how many Pods of each size
a cluster actually holds. Without that distribution there is no defensible way to combine the rows
into a single cluster-wide ratio, and none is offered here. The percentiles within each row are
meaningful; any statistic across rows would not be.

Within that limit, the rows support two statements. Compression is worth doing at every size class
measured — the weakest, at 3.5 KB, still removes 43% of the bytes, and the lowest ratio observed in
any individual Pod was 1.51×. And the ratio does not increase monotonically with size, so a ratio
cannot be extrapolated from an object's size alone.

Two further limits on what this establishes: it covers one resource on one cluster, and audit events
record mutations rather than the live object set, so these numbers speak to what a write costs, not
directly to database size. Both gaps are addressed by the analyzer described in the Test Plan.

### Goals

- Compress the serialized form of an API object before it is encrypted and written to etcd.
- Read pre-existing uncompressed values — encrypted or not — with zero migration, zero
  configuration, and zero ambiguity.
- Make disabling the feature safe without rewriting etcd.
- Make the at-rest format self-describing, so a heterogeneous fleet mid-rollout needs no
  coordination between apiservers.
- **Work whether or not encryption at rest is configured, through one code path and one at-rest
  format.** Encrypted clusters are the motivating case, because compression is the only mechanism
  that can reduce their stored size at all. Unencrypted clusters get the same benefit from the same
  code — there is no second implementation, no second format, and no configuration that selects
  between them.
- Per-resource opt-in from the first release, so an operator can compress ConfigMaps and custom
  resources without touching Secrets.
- Add no new third-party dependency.

### Non-Goals

- **Using compression to exceed the object size limit.** Enforced rather than merely declared:
  plaintext above the per-value ceiling is never framed, so such an object fails on write exactly as
  it does today.
- Compressing data in flight to clients — that is HTTP content negotiation, and already exists
  (KEP-2338).
- Hiding plaintext length. Padding into size buckets would close the compressibility side channel
  but negate the feature; see Risks and Mitigations.
- Cross-object or dictionary-based compression. This is a permanent, security-relevant non-goal.
- Compressing by default. The per-resource policy is empty at every stage including GA.
- Reducing watch-cache or apiserver heap usage.

## Proposal

Compression is configured by a file, pointed to by a single flag, and requires the
`StorageObjectCompression` feature gate. Absent the flag the feature is inert, which is how
"off by default" is expressed: a cluster that upgrades and takes no action stores byte-identical
values to before.

```yaml
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: StorageCompressionConfiguration
resources:
  # Ordered: the first entry matching a resource wins, exactly as in
  # EncryptionConfiguration. This is how a resource is excluded from a wildcard.
  - resources: ["events", "events.events.k8s.io"]
    algorithm: None
  - resources: ["*.*"]
    algorithm: Deflate
```

```
--feature-gates=StorageObjectCompression=true
--storage-compression-config=/etc/kubernetes/storage-compression.yaml
```

`algorithm` is `Deflate` or `None`. Resource entries use the same forms as
`--encryption-provider-config`: `<resource>`, `<resource>.<group>`, `*.<group>`, or `*.*`.

A file rather than flags, for three reasons. The ordering rule is what makes exclusion expressible at
all — a flat list of resources cannot say "everything except Events", and Events are a resource this
KEP explicitly considers an anti-target: high-churn, small, short-TTL and often on a separate etcd, so
compressing them is CPU spent for almost no durable saving. Second, the shape is borrowed rather than
invented, so an operator who has written an `EncryptionConfiguration` already knows how to read it.
Third, and decisively for an alpha feature: a flag is effectively permanent, inherited by every
generic-apiserver consumer through `EtcdOptions`, whereas a `v1alpha1` type carries no compatibility
promise and can be reshaped at beta once there is data to shape it with.

`secrets` is never selected by a wildcard. Compressing Secrets requires naming the resource
explicitly, which is a deliberate enough act to serve as the acknowledgement; doing so logs a warning
at startup naming the side channel described under Risks and Mitigations.

The configuration is read once at startup. There is no hot reload at alpha — deliberately, since a
per-resource storage format should not change from a file edit and a poll interval, and because
`--encryption-provider-config` sets the precedent that reload arrives later as a separate opt-in flag
if it is wanted at all.

Thresholds, compression level and inflation caps remain compile-time constants. There is no data yet
on which an operator could tune them, and the config file gives them a natural home to move into at
beta if the benchmarking justifies it.

Reads need no configuration. A value carries its own format, so any apiserver of this version or
newer reads any value written by any other, whatever its configuration.

### User Stories

#### Story 1: an encrypted cluster with no other lever

A platform operator runs envelope encryption via KMS, and most of their etcd content is custom
resources stored as JSON. Every etcd-side option is useless because etcd only ever sees ciphertext.
They name their CRDs in the compression config, restart the control plane, run a storage version
migration, and see the compacted database shrink.

#### Story 2: an unencrypted cluster

An operator with no encryption configured has a large number of ConfigMaps. They get the same
reduction from the same configuration and the same code path, with no security trade-off worth
arguing about, because plaintext length was already fully visible in etcd.

#### Story 3: an operator who wants compression but not on Secrets

An operator wants ConfigMaps and custom resources compressed but considers the compressibility of a
Secret's plaintext to be information their threat model cares about. A wildcard never reaches
`secrets`, so `*.*` gives them everything else without any carve-out on their part.

#### Story 4: an SRE during a rolling upgrade

Several apiservers are mid-rollout with different configurations against one etcd. Nothing breaks, in
any order, because every read is decided by the bytes rather than by that apiserver's configuration.
There is no "enable reads everywhere first, then enable writes" sequence to get wrong.

### Notes/Constraints/Caveats

**The ordering constraint drives the whole design.** Compression must be strictly inside encryption,
which rules out every insertion point above the transformer chain, and rules out doing this in etcd at
all.

**The intuitive design — a new on-disk prefix — does not work.** A `prefixTransformers` member would
have to wrap the encryption provider to get the ordering right, so a value on disk would carry a
compression prefix *and* an encryption prefix, with the compression prefix outside the ciphertext:

```
k8s:enc:deflate:v1:  k8s:enc:aescbc:v1:key1:  <ciphertext of the compressed plaintext>
└── compression ───┘ └──── encryption ──────┘ └──── encrypted ────────────────────────┘
```

That outer prefix buys nothing. The reader must decrypt before it can decompress, and by then it is
holding the plaintext, whose own first byte already answers whether the value is compressed — which is
information the format needs regardless, because during migration a single resource holds both
compressed and uncompressed values. So the prefix is pure addition on top of a mechanism that cannot
be removed. It also collides with `identityTransformer`, which rejects any value beginning
`k8s:enc:`, and it breaks existing assertions that a stored value carries exactly one provider prefix.
A secondary effect is that it would disclose in cleartext which resources are compressed, removing an
ambiguity that otherwise limits length-based inference — a refinement of an existing leak rather than
a new one.

**The `0x00` discriminator is an invariant about apimachinery that this KEP does not own.** It holds
because protobuf storage begins `6b 38 73 00`, JSON `{`, CBOR `d9 d9 f7`, YAML a printable key
character, and the legacy base64 fallback uses only base64-alphabet bytes. A test walks every storage
serializer and asserts it, and the writer escapes rather than rejects a leading `0x00` so the format
stays total even if the invariant were ever violated. It nonetheless deserves a note in apimachinery
rather than living only here.

**Small objects are deliberately excluded.** Below a 1 KiB threshold the ratio does not repay the CPU
spent on every read and write. As the fixture measurements show, this excludes most Secrets,
ConfigMaps, Leases, Roles, Events and Endpoints as they appear in minimal form.

**Enabling a resource causes one-time write amplification.** `GuaranteedUpdate` normally skips the
etcd write entirely when a client's update serialises to bytes identical to what was read — which is
why controllers re-applying an unchanged spec or status cost no etcd revisions. Flagging a value stale
deliberately defeats that comparison, because that is the mechanism by which a value migrates to a new
at-rest form. So once a resource is added to the configuration, idempotent controller updates that
were previously free each become one real write and one revision, until every object has been
rewritten once. It is bounded and self-terminating, but on a large cluster with chatty controllers it
is a visible burst, and is best driven deliberately with a storage version migration rather than left
to organic churn.

**Decompression is not disableable, by design.** This asymmetry is the central safety property and is
worth stating as a constraint rather than a feature: the format is a one-way commitment. Once a
release can write a frame, every subsequent release must be able to read one, forever.

### Risks and Mitigations

**Compressing before encrypting leaks plaintext compressibility.** This is the significant security
cost and the main thing a sig-auth reviewer should scrutinise.

Ciphertext length is already a function of plaintext length for every provider, so exact plaintext
length is *already* exposed to anyone who can read etcd or a backup. Compression converts a leak of
length into a leak of compressibility, which is a coarse proxy for entropy and internal repetition.
CRIME- and BREACH-style attacks additionally require chosen plaintext compressed *in the same context*
as a secret, plus a repeatable length oracle. Cross-object attacks are structurally impossible here:
every value is compressed independently with a freshly reset compressor, with no shared dictionary and
no cross-value state — an invariant recorded in code precisely because the obvious future optimisation
would destroy it.

The residual risk is intra-object, and it requires three conditions at once: an adversary who can write
part of an object, cannot read that object, and can observe its stored length. The third narrows it
sharply. Observing stored length means reading etcd or a backup, and in an unencrypted cluster such an
adversary already reads the plaintext directly, so the oracle buys nothing; the channel therefore has
teeth only in an encrypted cluster, where ciphertext lengths are visible but plaintext is not. The
second is usually vacuous — not because write implies read, since RBAC verbs are granted independently,
but because they are conventionally granted together. Two shapes express it: `update` or `patch` on a
resource without `get` or `list` on it, and `patch` on a subresource such as `pods/status` without `get`
on the parent. Mitigations: no resource is compressed unless the configuration selects
it; `secrets` is never reachable by wildcard, so compressing it requires naming it explicitly and logs
a warning that names the channel; and observability is aggregate-only, with no per-object ratio or
byte-count metric, because those would be a finer oracle than the length channel itself. Length
padding would close the channel and is rejected as negating the feature. Residual risk is accepted and
stated; a sig-auth review of exactly this residual is an alpha graduation requirement.

**Decompression bombs.** `aescbc` and the KMS v1 CBC read path are unauthenticated and malleable, and
`identity` does not authenticate at all, so an actor able to tamper with etcd can steer bytes into the
decompressor without holding any key. This is the first apiserver code path that allocates based on
length metadata read from storage, so the bound is explicit rather than argued: the declared length is
checked against a cap before anything is allocated, there is exactly one right-sized allocation, the
stream is bounded in both directions independently of its own declaration, and concurrent inflations
are capped process-wide. A fuzz target is mandatory rather than optional.

**Rolling back below the first decompress-capable release breaks the cluster.** An older apiserver
decrypts a frame successfully and then fails to decode it. One undecodable value aggregates into a
`StatusReasonStoreReadError` for a whole LIST prefix and tears down watches, which for the watch-cache
reflector removes cached serving for that resource. This is an outage, not a degradation. Mitigations:
writing a frame is never reachable by default at any stage, requiring both the gate and a configuration
file that names a resource, so no cluster acquires frames without a deliberate operator action; and the
only sound pre-downgrade procedure is a completed storage version migration per compressed resource,
because no read-derived metric can prove that no compressed value remains — a cold object is never read
and so never observed. See [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

**Version skew must not be mistaken for corruption.** A frame carrying an algorithm this binary does
not implement was written by a newer apiserver and is perfectly readable by one. Classifying it as a
corrupt object would surface it to clients as a `CorruptObjError`, which is the signal that invites an
operator to reach for `kubectl delete --ignore-store-read-errors-with-cluster-breaking-potential` and
destroy undamaged data. Two guards: such errors are excluded from corrupt-object classification, and
the storage layer refuses the read outright so the unsafe-delete path cannot proceed regardless of the
option.

**CPU on the write and read hot paths.** Compression sits on every mutating request for a selected
resource, and the worst read case is watch-cache initialisation — a full LIST decompressed
sequentially on the request goroutine. KEP-2338 measured the analogous mistake directly: enabling gzip
for all suitable API responses "caused a significant performance regression in both CPU usage (2x) and
tail latency (2-5x) on the Kubernetes apiservers", which is why a 128 KB size floor was introduced
([KEP-2338 README][kep-2338], lines 65-69 and 82-85). Mitigations here: level 1, a 1 KiB floor, pooled
coder state, per-resource opt-in, and DEFLATE's asymmetry — decompression performs no match search,
only Huffman decoding and byte copies, so on real Pods it runs 1.7-2.0x faster than level-1 compression
and up to 29x faster than level 9, the gap widening with level because level affects only the encoder
(see [the level appendix](#appendix-measuring-the-compression-level-trade-off) for the measurements and
method). A watch-cache-initialisation benchmark is an alpha deliverable.

**APF could under-charge LIST memory.** The resource size estimator records stored bytes, and that
figure bounds the memory API Priority and Fairness believes a LIST will occupy — but the memory
actually consumed is the decompressed, decoded size. Left uncorrected, a resource compressing 5x would
let APF admit roughly 5x more concurrent LISTs than the apiserver can hold. The estimate is therefore
scaled by an observed expansion factor, floored at 1.0 so it can only ever be more conservative than
the raw measurement.

**Raw-etcd tooling sees an opaque value.** `auger`, `etcdhelper` and snapshot-based forensics parse
etcd values directly and are the standard answer to "what is in this key?". In unencrypted clusters a
frame makes them fail, and none of them can be fixed by a kubernetes/kubernetes change. An `auger`
change is a beta graduation dependency, and the frame carries a literal `k8s` at bytes 1-3 so a human
reading a hex dump can at least recognise what they are looking at.

[kep-2338]: https://git.k8s.io/enhancements/keps/sig-api-machinery/2338-graduate-API-gzip-compression-support-to-GA/README.md

## Design Details

### The at-rest byte format

A framed value is a 5-byte fixed header — a 4-byte magic and one algorithm byte — followed by a body
whose shape depends on the algorithm:

```
offset  size  value / meaning
------  ----  ----------------------------------------------------------
0       1     0x00        discriminator; no storage serializer emits it
1       3     6b 38 73    "k8s"; structure check, and recognisable in od
4       1     algorithm   algStored (0x00) | algDeflate (0x01)
--- algStored ---
5       n     plaintext, verbatim                    overhead: 5 bytes
--- algDeflate ---
5       w     declared plaintext length, LEB128 (1-3 bytes)
5+w     m     raw DEFLATE stream (RFC 1951)          overhead: 6-8 bytes
```

The magic is `0x00` followed by the ASCII `k8s`, which is protobuf's own storage prefix `6b 38 73 00`
rotated so the NUL leads — a coincidence, but a helpful one when reading a hex dump, since `k8s` still
appears in reading order.

Where this sits at rest depends on the encryption configuration, and the unencrypted case is the one
worth stating first because it is both the default and the most exposed:

- **No `--encryption-provider-config`.** The transformer is `identity.NewEncryptCheckTransformer()`,
  which writes the serialized object through unchanged and with *no prefix at all*. So today the value
  in etcd simply *is* the serialized object, and with compression enabled it becomes the frame
  directly: the leading `0x00` is literally the first byte in etcd. This is also the only configuration
  with no integrity check anywhere beneath us; see [Integrity](#integrity-what-authenticates-a-frame).
- **Encryption enabled.** The frame is the plaintext that the encryption transformer consumes, so it is
  invisible at rest, wrapped inside a provider prefix and ciphertext. Those prefixes follow the grammar
  `k8s:enc:<provider>:<providerVersion>:<name>:` — `aescbc`, `aesgcm` and `secretbox` are at `v1` and
  append the key name, while `kms` exists at both `v1` and `v2` and appends the provider name, giving
  real-world values such as `k8s:enc:kms:v2:aws-encryption-provider-v2-aok:`.

A `deflate` frame declares its plaintext length so the read path can bound its allocation before
inflating anything; see [Bounding decompression](#bounding-decompression). A `stored` frame needs no
such field, since its body length *is* its plaintext length.

The frame carries no version byte, no flags byte, no reserved byte and no compression level. The
algorithm id serves as the version discriminator, because an unknown id already conveys exactly what a
reader needs to know — that a newer apiserver wrote this — and a future format change simply claims a
new id. The level is omitted because it is a writer-side choice the decoder never needs: any DEFLATE
stream decodes without knowing how it was produced, which is also what would let a per-resource level
be added later without a format change.

**Why the overhead is affordable, and why the size floor exists.** Framing costs 5 bytes for `stored`
and 6-8 for `deflate` — at worst 0.78% of a 1 KiB object and 0.003% of a 256 KiB one. Below roughly a
kilobyte that stops being negligible while the gain stops being worth having: a few-hundred-byte object
such as a Lease (485 bytes as an in-tree fixture, compressing only 1.42x) saves too little to repay the
CPU spent on every read and write of it, and a genuinely incompressible one would simply grow. `MinPlaintextBytes = 1024` keeps the overhead off those objects entirely, since they are
never framed and so never pay it.

### Why the ordering is not configurable anywhere

Nothing in this design configures the order of compression and encryption. A decorator's two methods
run on opposite sides of the value it wraps: on write, code before the delegate call sees plaintext and
code after it sees ciphertext; on read the delegate runs first, so the wrapper sees whatever the
delegate produced.

```go
func (t *transformer) TransformToStorage(ctx context.Context, plaintext []byte, dc value.Context) ([]byte, error) {
    framed := t.frame(plaintext)                                 // sees plaintext
    return t.delegate.TransformToStorage(ctx, framed, dc)        // ... then encryption
}

func (t *transformer) TransformFromStorage(ctx context.Context, data []byte, dc value.Context) ([]byte, bool, error) {
    plaintext, stale, err := t.delegate.TransformFromStorage(ctx, data, dc)  // decryption first
    // ... then unframe
}
```

Compression must see plaintext to be worth anything, so on write it can only go before the delegate
call — which *is* compress-then-encrypt — and the read method is then necessarily
decrypt-then-decompress. There is no sequencing table to get wrong and no way to reach
encrypt-then-compress, which would yield a ratio near 1.0 because ciphertext is indistinguishable from
random. This is the same composition `prefixTransformers` already uses, so it adds no new concept to
the storage stack.

### The write path

A single predicate, `willFrame(plaintext)`, decides whether to emit a frame. It is a pure function of
the plaintext: its length against both the floor and the per-value ceiling, whether it begins with
`0x00`, and whether the policy selects the resource.
Which algorithm the frame then carries is a separate question:

| plaintext | policy selects resource, 1 KiB to 1.5 MiB | policy excludes resource, or size outside that band |
| --- | --- | --- |
| begins with `0x00` | `stored` — the escape, checked first | `stored` — the escape |
| anything else | `deflate` (or `stored` if it does not shrink) | stored bare, no frame |

**The escape.** No Kubernetes storage serializer emits a leading `0x00`; a test walks every one of them
to pin that. The escape is not there for a caller we know about, it is there so the read side is
correct by construction rather than by assumption. `unframe` decides "this is a frame" by testing the
first byte, and that test is exact only if we frame *every* plaintext beginning with `0x00`. Were one
ever to arrive — a serializer change, a third party vendoring `k8s.io/apiserver` and driving
`TransformToStorage` with its own encoder, a fuzzer — and were we to store it bare, then on read
`isFramed` would return true for something that is not a frame and `unframe` would either fail or
return garbage: silent corruption caused by a change three layers away.

The escape is tested *before* the policy and the size floor, and it always emits `stored`, so a
`0x00`-leading plaintext is never deflated even when the policy selects its resource and it exceeds the
floor. Compressing it instead would be equally correct, and would save bytes in the hypothetical where
such a plaintext is also large and compressible. It is not worth the second code path: the branch is
one a test asserts is unreachable, so its value lies entirely in being simple enough to reason about
without testing it against live data. One consequence to note is that a `stored` frame produced by the
escape never migrates back to a bare value, because `willFrame` keeps returning true for that plaintext
whatever the policy says — which is precisely the property that stops the rewrite loop discussed below.

**`stored` is a memo, not a fallback.** When DEFLATE fails to shrink a payload we still emit a frame,
with `alg = stored` and the body copied verbatim. The algorithm byte is best understood as a one-byte
note recording *"we already looked, compression did not help, do not look again."* That is what makes
the next section work.

### Staleness, and why framing must not depend on the compression outcome

`willFrame` is also the staleness predicate. On every read the layer compares how a value is stored
against how it should be stored, and reports `stale` when they disagree, which causes
`GuaranteedUpdate` to rewrite it on the next update. Using literally the same function at both sites is
what makes the migration terminate.

It is tempting to make framing depend on whether compression actually helped — "frame only if it
shrinks, ties go bare." That rule is perfectly well-defined and deterministic: DEFLATE at a fixed level
is a function, so the same plaintext always yields the same answer, and it does not oscillate. The
problem is the cost of *evaluating* it on the read path. Under a length-and-policy rule the staleness
comparison is two integer tests. Under a ratio rule it requires compressing the plaintext — 38 µs to
1.1 ms per object, per [the appendix](#appendix-measuring-the-compression-level-trade-off) — on the path
that is orders of magnitude hotter than writes and that fans out per object on a LIST served from etcd.

Nor can the check be skipped. If a bare value is never examined, objects that would compress well never
migrate when the policy is switched on. If instead it is approximated with "bare + selected + ≥ 1 KiB
implies stale," then every read of a genuinely incompressible object reports stale, the rewrite produces
byte-identical bare output, and the next read reports stale again: a rewrite loop that never converges
even though the stored bytes never change.

Framing unconditionally on length and policy, with `stored` recording the outcome, removes the dilemma.
"Is it framed?" and "should it be framed?" are answerable from the same two cheap facts, so the check
is exact and O(1), and a stale value converges in exactly one rewrite. The price is that an
incompressible object of at least 1 KiB grows by the header — 0.5% at the threshold, less above. That
is the cost of a terminating migration.

### The read path

Decompression is unconditional and format-driven. The decorator is installed whatever the feature gate
and configuration say, because read support must never depend on configuration: a value framed by any
apiserver has to be readable by every apiserver of that version or newer under any flags, or dropping a
flag would strand data. Default inertness comes from consulting the policy inside `TransformToStorage`,
and from `unframe` returning non-framed bytes verbatim.

One property of RFC 1951 is worth recording because it bounds what the format can promise. A DEFLATE
stream is a sequence of blocks, each carrying a 1-bit `BFINAL` header, and the decoder stops the
instant it finishes a block with `BFINAL` set. Bytes appended after that block are never examined:
`flate.Reader` returns the correct plaintext and `io.EOF` and never reports them. So `unframe` cannot
detect trailing junk, and a frame with bytes appended still decodes to the right object. Two
consequences: the frame is **not** canonical, so byte-identity must never be used as a staleness signal
(it is not); and an attacker who can append to a stored value cannot change the decoded object, which
means the corruption vector of interest is bit-flips *inside* the stream rather than appends.

### Bounding decompression

This is the first apiserver path that allocates based on length metadata read from storage, so the
bound is explicit rather than argued. The declared output is checked against `MaxDecompressedBytes`
before anything is allocated; there is exactly one right-sized allocation; the stream is bounded in
both directions independently of its own declaration; and concurrent inflations are capped
process-wide by a semaphore acquired with the request context, so a caller that gives up stops waiting.

The cap is currently derived rather than configured: `maxInflightInflations()` sizes the semaphore at
`min(max(GOMAXPROCS*2, 4), 64)`. **A `maxConcurrentDecompressions` field on the configuration file is
proposed for beta**, alongside the size floor and ceiling (see
[Graduation Criteria](#graduation-criteria)), so an operator can lower it on a memory-constrained control plane or raise it on a
read-heavy one, since no single formula suits both. Two properties of that field should be disclosed
rather than hidden. It would be **process-scoped** while `resources` is per-resource, because the
semaphore protects process memory rather than any one resource. And because the read path must stay live
with the gate off and with no configuration file present, the override can only be applied where the
file is parsed, so with the gate off the derived default continues to apply — safe, since reads keep
working, but not tunable.

The memory ceiling either way is `n × (40 KiB + MaxPlaintextBytes)`, approximately `n × 1.54 MiB`, or
~98 MiB at the current effective maximum of 64, and only if every concurrent inflation is
simultaneously a maximum-size object. A count is the natural unit because that is what the semaphore
admits, but that byte figure is what an administrator should size against; see
[the memory appendix](#appendix-measuring-compressor-and-decompressor-memory-footprint).

### Integrity: what authenticates a frame

Whether a corrupted frame is caught before it reaches the decompressor is entirely a property of the
layer beneath, and what matters is whether that layer authenticates on *read*:

- **Authenticated Encryption with Associated Data (AEAD)** — `aesgcm`, `secretbox` and `kms` v2 — stores
  a tag over the ciphertext and rejects any modification with overwhelming probability before releasing a
  single plaintext byte. Corruption never reaches `unframe`.
- **`aescbc`, the `kms` v1 read path, and no encryption at all** offer *identical* guarantees, namely
  none. Neither stores a tag, MAC or redundancy that a reader will refuse to look past. `kms` v1 belongs
  here despite writing AES-GCM: its base transformer is a union of GCM and AES-CBC, kept so that values
  written before v1.25 remain readable, and a value that fails GCM authentication is retried under bare
  CBC — whose output then reaches `unframe`.

AES-CBC deserves a note because "unauthenticated" understates it. CBC pads the plaintext, prepends a
random IV, and XORs each block with the previous *ciphertext* block before applying the block cipher.
That provides confidentiality only: decryption always succeeds, since any 16-byte-aligned input
decrypts to something, and the sole integrity signal is PKCS#7 unpadding, which rejects
roughly 255 of every 256 corruptions that randomise the final block, since a random block carries valid
padding with probability about 1/256 — and detects nothing at all elsewhere. It is also malleable in a structured
way: flipping bit *i* of ciphertext block *n* flips exactly bit *i* of plaintext block *n+1* while
randomising block *n*, giving an attacker a chosen-bit-edit primitive at the cost of destroying the
preceding block. This is why upstream documents `aescbc` as not recommended.

Compression changes the shape of undetected corruption for those two configurations, measurably. Across
4000 single-bit flips into a 287 KB Pod in protobuf storage form (ratio 6.0):

| | uncompressed | DEFLATE frame |
| --- | --- | --- |
| caught by inflate | — | 4.97% |
| caught by protobuf decode | 4.50% | 67.10% |
| **total detected** | **4.50%** | **72.08%** |
| decodes, silently wrong object | 95.50% | 27.93% |
| mean corrupted plaintext bytes when silent | **1.0** | **184.7** |

Compression improves detection roughly 16-fold, but a flip that does slip through corrupts ~185 bytes
rather than 1, because a wrong back-reference distance or length corrupts everything downstream.
Expected silently-corrupted bytes per flip therefore rises from 0.955 to 51.6 — a ~54x regression
despite the better detection rate. Whether to close that with a checksum is an open question for
reviewers rather than a decision this KEP makes; see [Open Questions](#open-questions).

### API Priority and Fairness, and size accounting

**How APF accounts for LIST size today.** With `SizeBasedListCostEstimate` enabled, LIST cost comes
from `seatsBasedOnObjectSize` rather than from counting objects. It multiplies `objectsLoadedInMemory`
by `stats.EstimatedAverageObjectSizeBytes` to get `memoryUsedAtOnce`, then charges
`ceil(memoryUsedAtOnce / bytesPerSeat)` with `bytesPerSeat = 100_000` — one seat per 100 KB the LIST is
expected to hold at once. `objectsLoadedInMemory` is 1 for a request naming a single object,
`max(limited, ObjectCount/2)` when a field or label selector is present, and `limited` otherwise.
Cache-served LISTs are additionally clamped to `cacheWithStreamingMaxMemoryUsage = 1_000_000`, so at
most 10 seats. And `EstimatedAverageObjectSizeBytes` comes from the resource size estimator, which
averages `len(kv.Value)` — the **stored** size.

**The gap.** `len(kv.Value)` is the compressed, encrypted size, but the quantity APF wants is the
memory the LIST will occupy, which is bounded below by the plaintext. At a 6x ratio APF would charge a
sixth of the seats it should and admit roughly 6x the concurrent LISTs the apiserver can hold. Usefully,
the clamp means this only bites on LISTs delegated to etcd — which is also exactly the path that
performs the decompression, and therefore the only client-drivable amplification vector in this design.
The watch path is not one: the watch cache opens a single etcd watch per resource and decodes each
event once before dispatching the decoded object to every client watcher, so a client cannot create
etcd watches and inflation there is bounded by the resource's write rate.

**The correction.** The estimator scales its average by an expansion factor the transformer observes on
reads — an EWMA over actual (plaintext, stored) pairs with α = 1/64 — floored at 1.0 so the result can
only ever be more conservative than today's number. This is plumbed as a `PlaintextSizeExpander`
interface, which the corrupt-object error-interpreting transformer must explicitly forward, since
embedding `value.Transformer` promotes only that interface's methods and the estimator would otherwise
silently lose the correction whenever unsafe-delete support is enabled. A residual remains and is out
of scope: the expansion from plaintext to decoded object graph was already unaccounted for before this
KEP.

### Error classification and version skew

Two sentinels are deliberately classified as *not* corruption: an unsupported algorithm, and a
malformed frame. A frame carrying an algorithm this binary does not implement was written by a newer
apiserver and is perfectly readable by one; a malformed frame is more likely a bug in this layer than
damaged bytes. Classifying either as a corrupt object would surface it to clients as a
`CorruptObjError`, which is precisely the signal that invites an operator to reach for `kubectl delete
--ignore-store-read-errors-with-cluster-breaking-potential` and destroy an object that was never
damaged. Two guards apply: both are excluded from corrupt-object classification, and the store refuses
the read outright so the unsafe-delete path cannot proceed regardless of the option. One consequence to
state plainly: the LIST error aggregator treats anything that is not a corrupt-object error as a reason
to abort the whole list rather than to aggregate per item, so one such value fails an entire LIST. That
is the intended trade — one loud, unambiguous "upgrade this apiserver" beats many partial ones.

**GOAWAY was considered for the skew case and rejected.** A filter could set `Connection: close` on
such a response, as `WithProbabilisticGoaway` does, prompting the client to reconnect and possibly land
on a newer apiserver. Four reasons not to. It offers no affinity: GOAWAY redistributes across whatever
the load balancer offers and cannot ask for an apiserver implementing algorithm *N*, so mid-upgrade the
retry is a coin flip with no mechanism to converge. It makes a skew violation *quieter*, and this
condition can only arise from a skew wider than supported or from an algorithm shipped without an N-1
read gate — operator-visible faults that should fail identically on every attempt rather than become
retry-eventually-succeeds. Content-dependent connection teardown is a foot-gun: the header would be set
as a function of a stored object's bytes, so one unreadable object in a large resource would tear down
every connection that lists it, repeatedly, across every client. And it does not help, because a LIST
is already all-or-nothing here; GOAWAY changes which server the *next* attempt hits, not whether this
one works. The actual mitigations are that the alpha algorithm set is exactly `stored` and `deflate`,
so within a supported skew window the error is unreachable, and that any future algorithm must ship
read support one release before any release may write it.

### Observability

Five ALPHA metrics. `compression_operations_total{resource,operation,outcome}` distinguishes values
compressed from values stored verbatim from values not framed at all, so an operator can tell whether
the size floor or the policy is the reason a resource is not being compressed.
`compression_duration_seconds{operation}` exists because compression time is otherwise invisible: the
etcd latency tracker wraps only the clientv3 round trip, the store's tracing spans emit only above
500 ms, and because this layer sits outside `prefixTransformers` its cost is absent from
`apiserver_storage_transformation_duration_seconds`.
`compression_rewrites_needed_total{resource,direction}` bounds and attributes the write amplification
of enabling or disabling a resource, and goes quiet once a migration converges.

`compression_unsupported_algorithm_total{resource}` is labelled and is not pre-initialised, so a child
series is created only on first increment — a healthy cluster carries zero series for it, which is
strictly cheaper than an unlabelled counter sitting at zero forever, and a broken cluster gets exactly
the label values it needs. Neither counter is client-drivable, since both arise from stored bytes rather
than from request content, so there is no cardinality attack.

`compression_format_errors_total{resource}` carries the resource and, like the counter above, is not
pre-initialised. What it deliberately omits is the *reason* a frame was rejected, and that omission is
not about cardinality. A label carrying the reason — bad magic, invalid declared length, stream shorter
than declared, stream longer than declared — would be a chosen-ciphertext format oracle against the
providers that do not authenticate on read: an actor able to tamper with etcd could iterate candidate
ciphertexts and learn which validation step failed by reading `/metrics`, without holding any key. That
distinguishing detail is logged at v4 instead, where an operator can see it and a `/metrics` reader
cannot. The resource carries no such signal; the reason is the part that must never become a label.

No ratio or byte-count metric is exported. On a resource holding very few objects the `outcome` label
already makes one property of each write observable to whoever can read `/metrics`: whether the object
crossed the framing size floor. That is not a new capability — reading `/metrics` is an
already-privileged cluster-scoped grant, and object length is exposed more coarsely through
`apiserver_storage_size_bytes` and the resource size estimate — and it is narrower than it looks, since
`stored` exists and so the label reflects only the size floor and the `0x00` escape, never
compressibility. A per-resource ratio would be a genuinely finer content-dependent signal, which is why
there is not one.

### Test Plan

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

**The `0x00` discriminator invariant needed a test before it could be relied on.** The whole
backward-compatibility argument rests on no kube-apiserver storage serializer ever emitting a leading
`0x00`, and nothing in the tree asserted it. A test now drives every storage serializer — protobuf,
JSON, YAML, CBOR and the legacy base64 fallback — over a range of object shapes, including shapes that
deliberately push `0x00` into field content rather than into framing, and requires the read path to
classify each encoding as unframed and return it byte-identically. If that test ever fails the format
needs a new discriminator; it must not be weakened. A companion test restating the known serializer
lead bytes as literals documents the assumption but cannot detect a change to it, since it does not
reference the serializers' own constants; wiring it to them, or deleting it in favour of the serializer
walk, is one of the gaps listed under Unit tests below.

**No existing raw-etcd or encryption test needed changing, and that is itself the check.** The tests
that read values straight out of etcd for every resource in the tree, and those that assert the
encryption envelope's prefix byte for byte, all pass unmodified even though the decompression path is
installed unconditionally. Nothing in that corpus was relaxed to accommodate this feature, and any
future change that would require relaxing one should be treated as a design regression rather than as
test maintenance.

**Two existing components changed and carry their own new tests:** the resource size estimator, which
now scales its average by an observed expansion factor, and the corrupt-object error wrapper, which
must forward that factor explicitly or the correction is silently lost whenever unsafe-delete support
is enabled.

**The encryption integration harness cannot yet express "compression on, encryption off", and teaching
it to is a prerequisite.** It requires an encryption configuration today, so it cannot start an
apiserver with a compression configuration and no encryption provider. That is the case most worth
testing, being both the default deployment and the one where the frame is the first byte in etcd.

##### Unit tests

Coverage of the packages this KEP touches, from local `go test -cover` runs at HEAD rather than from
the [coverage job](https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit) the
template points at:

- `k8s.io/apiserver/pkg/storage/value/compression`: `2026-08-28` - `91.7%`
- `k8s.io/apiserver/pkg/storage/etcd3`: `2026-08-28` - `79.4%`
- `k8s.io/apiserver/pkg/apis/apiserver/load`: `2026-08-28` - `89.5%`
- `k8s.io/apiserver/pkg/apis/apiserver/validation`: `2026-08-28` - `87.7%`
- `k8s.io/apiserver/pkg/storage/storagebackend/factory`: `2026-08-28` - `53.8%`
- `k8s.io/apiserver/pkg/server/options`: `2026-08-28` - `50.4%`

The last two need reading with care, and only one means what it appears to. The `server/options` figure
does include this feature's flag validation, which is itself fully covered. The
`storagebackend/factory` figure does not: the only tests exercising the configuration loader live in
another package, and per-package coverage gives no credit to cross-package callers, so the loader's
rejection of a fully shadowed entry, its warning when `secrets` is named, and its per-path caching are
all effectively uncovered by the number quoted. Two further packages carry only types, defaults and
generated conversions and have no test files at all; both are exercised indirectly.

What the unit tests establish, stated as properties:

- **The at-rest format is pinned as a golden value**, byte for byte, in the same spirit as the existing
  AES transformer's stability test, so an accidental change to the header, the algorithm ids or the
  declared-length encoding fails loudly rather than silently altering what is already on disk.
  Round-trip fidelity, tolerance of trailing bytes, and the exact boundary sizes are covered alongside.
- **The read path is total.** Every malformed input is classified as either a format error or an
  unsupported algorithm, never as a panic and never as a partial result, and the inflated output is
  bounded regardless of what the frame declares. Two fuzz targets carry this: one over the unframing
  routine, one over the whole transformer round trip. Both matter because the unauthenticated providers
  let an actor who can reach etcd choose the bytes that arrive here.
- **Framing and staleness agree**, which is the property that makes migration terminate. The predicate
  deciding whether to frame a value is the same one deciding whether a stored value is stale, a stale
  value converges in exactly one rewrite, and a plaintext that took the ambiguity escape is never
  reported stale. This failure mode surfaces only as unbounded etcd revision growth, so it is tested
  directly rather than left to observation.
- **Ordering holds against a real encryption chain**, not a stub: a value is compressed and then
  encrypted, and unsealing the stored bytes shows the frame inside the ciphertext.
- **The feature is inert by default** — no framing, no metrics and no allocation when no resource is
  selected — and never expands a value beyond the framing overhead.
- **Policy resolution behaves as specified**: first match wins, the wildcard forms expand as documented,
  `secrets` is unreachable by wildcard and must be named explicitly, a fully shadowed entry is
  rejected, and entry order survives loading and defaulting. The operator-facing error text is asserted
  for each rule, along with the two inputs that must be *accepted*: cross-entry overlap where the
  earlier entry is more specific, and a resource name the encryption configuration's validator rejects
  but this one must not.
- **Format and skew errors are not corruption.** Both survive error aggregation intact, a LIST
  containing one aborts rather than aggregating per item, and an unsafe delete against such an object is
  refused while a genuinely undecryptable object stays deletable. Concurrency is exercised under
  `-race`.
- **APF accounting is corrected end to end**: the expansion factor starts at one, converges upward, is
  floored at one so it can only be conservative, and survives the corrupt-object wrapper that would
  otherwise drop it.
- **Metrics expose exactly what is intended**, asserted against exact exposition text: every write and
  read outcome, no samples at all when the feature is inert rather than a row of zeros, and for the
  format-error counter, that it carries the resource but no label describing why a frame was rejected,
  with a child series appearing only once an error has been counted.

**What is not yet covered and needs writing**, each a real hole rather than a rounding error: an
ambiguous plaintext at or above the size floor on a selected resource, both existing cases sitting
below it; in-package tests for the configuration loader, per the coverage note above; the proposed
concurrency limit, which needs a field, validation and a plumbing test; an allocation assertion on the
inert read path, since the existing benchmark measures only time; a committed fuzz corpus, to which any
alpha crasher belongs, with a reserved algorithm byte added to the seeds; a direct table test for
configuration validation, reached today only indirectly; a test for the compression latency histogram,
which has none; and a guard on the writer memory footprint, whose measurement harness is not committed; and wiring the
literal lead-byte companion test to the serializers' own constants, or deleting it in favour of the
serializer walk.

##### Integration tests

There are none today; all of the following are alpha deliverables, gated on the harness change above.
Most belong beside the existing encryption transformation tests, which already start a real apiserver
against a real etcd with arbitrary flags and can read and write raw etcd records.

- **Round trip in both encryption states, including a JSON-stored custom resource.** With encryption
  on, the raw value still begins with the provider prefix byte for byte, and unsealing it shows a frame
  inside; with no encryption provider the raw value *is* the frame. Either way the object reads back
  identically through the API.
- **Migration and staleness convergence**, which also substantiates the write-amplification claim:
  create objects with the policy off, restart with it on, LIST once and watch the rewrite counter rise,
  update each object, then require every raw value framed and a second LIST to leave the counter
  unchanged — then the same in reverse. The existing migration tests drive a similar shape through key
  rotation but cannot introduce staleness by restart, so this test comes first and the
  migration-driven version follows it.
- **Inertness and startup validation.** With the gate off and a configuration file present, startup
  must fail with an actionable message; with the gate off and no file, raw etcd bytes must be unchanged.
  A malformed configuration must be fatal at startup, covering a fully shadowed entry, an unsupported
  algorithm and a missing path.
- **Version skew.** Plant a frame carrying an unimplemented algorithm and require that a GET fails
  without reporting corruption, that a LIST over the prefix fails outright rather than aggregating, that
  the skew counter increments, and that an unsafe delete is refused and the object survives. Under a
  real encryption provider the frame has to be sealed with the same key first, or it fails at decryption
  and never reaches this layer.
- **Corruption under an unauthenticated provider**, aimed rather than sampled, since the outcome
  distribution in [Integrity](#integrity-what-authenticates-a-frame) would make a random bit flip flaky.
  Then assert the cost: because a format error is deliberately not corruption, the unsafe deleter
  refuses it, so a genuinely damaged object becomes undeletable by the mechanism built for that
  situation. That is current behaviour and is the evidence feeding Open Question 1, since a checksum
  mismatch would need a third classification that *is* eligible for unsafe deletion.

**Benchmarks as gating artifacts.** Alpha numbers are reported rather than gated, with two exceptions
that do block: the safety bounds of tier B5, which are correctness properties with numbers attached,
and B1's inert-path allocation count.

- **B1 — micro-benchmarks**, per-PR and CI-runnable: did this change regress the hot path? In tree.
- **B2 — realised ratio on real data**: is the feature worth shipping? An offline analyzer over an etcd
  snapshot; not yet written.
- **B3 — storage-layer throughput against a real etcd**: what does an operator pay? Write throughput,
  LIST, stats and watch-cache initialisation across provider, policy, media type and size.
- **B4 — control-plane load**: does this hold under realistic concurrency? Scaled down at alpha, with
  the full-scale run deferred to beta.
- **B5 — adversarial and migration steady state**: are the failure modes bounded?

B2 reports distributions rather than means — per resource, the object count, value-size percentiles,
ratio, bytes saved, the fraction below the size floor and the fraction stored verbatim — keeps live-value
reduction separate from database-file reduction, which MVCC history and free pages confound, emits
aggregate statistics only and never object content, and must be runnable by an operator against their
own snapshot. B3 extends the existing storage-layer and encryption-provider benchmarks and adds
watch-cache initialisation as one more axis to a benchmark that already seeds 150 000 Pods into a real
etcd; any database-size cross-check needs an explicit defrag first, since the reported size is
allocated file size. B5's highest-value assertion is APF admission: compress a highly compressible
resource, drive concurrent LISTs to the admission limit, and require resident memory to stay inside the
uncompressed envelope. Then a decompression-bomb LIST bounded by the inflation limit, never-expand
under load, write amplification measured as revision growth, and the half-migrated steady state.

##### e2e tests

- N/A — no new e2e test code at alpha or beta. The beta artifact is a test *configuration*, not a test.

The reason is that this feature has no API surface: no endpoint, no object field, no request parameter
and no `kubectl` behaviour. The instructive precedent is the storage version migration of KEP-4192,
which does have an e2e test — but that test exercises create, update, list, patch and delete on the
`StorageVersionMigration` resource, because that resource is the feature's API surface. Whether a
migration actually rewrote anything in etcd is verified in integration tests instead. Compression has no
corresponding API surface, so the e2e half of that split is empty and the behavioural half is what the
integration tests above cover.

Nor is e2e the right layer for inspecting at-rest bytes, which is where every assertion worth making
lives. No e2e test in the tree reads the control plane's etcd; the framework's remote-command helper
targets nodes rather than control-plane hosts and is provider-gated; and on a cluster whose control
plane is not exposed — which includes much of the audience for a conformance suite — it is not reachable
at all. This is a question of the appropriate layer rather than an absolute impossibility: a self-hosted
cluster could be contrived to allow it, at the cost of a fragile, provider-specific test duplicating an
integration test.

What e2e can usefully do, and what beta should do, is demonstrate the absence of a regression. Objects
round-tripping unchanged is already asserted by every existing e2e and conformance test — but only on a
cluster configured for compression, and no default cluster is. Hence a beta job that runs the existing
suite unchanged against a cluster with the gate enabled and a policy covering ConfigMaps and a custom
resource, whose pass criterion is that nothing changes. Requiring particular apiserver configuration is
ordinary for such a job, which is what feature-tagged suites and their dedicated CI jobs exist for.

Four release-signoff sub-items consequently have no applicable content, three under the test plan and
one under graduation criteria:

- `e2e Tests for all Beta API Operations (endpoints)` — there are no endpoints.
- `(R) Ensure GA e2e tests meet requirements for Conformance Tests` — nothing is enabled by default at
  GA, so there is no behaviour a conformance test could assert on an unconfigured cluster.
- `(R) Minimum Two Week Window for GA e2e tests to prove flake free` — the beta job above supplies this
  if reviewers judge it in scope.
- `(R) all GA Endpoints must be hit by Conformance Tests within one minor version of promotion to GA` —
  again, no endpoints.

Three carry (R), so the determination is not the author's to make and needs explicit SIG Architecture
and SIG Testing sign-off recorded here. The graduation evidence is the fuzz corpus, the discriminator
invariant, the convergence and skew integration tests, and the B2–B5 measurements.

### Graduation Criteria

**Compression is never on by default, including at GA.** No resource is compressed unless an
administrator writes a configuration file and points `--storage-compression-config` at it; compressing
by default is an explicit Non-Goal. The stages therefore move *reachability*, not activation:
`StorageObjectCompression` is alpha and off by default at v1.38, beta and on by default at v1.39 while
still doing nothing without a configuration file, then locked on at GA. A reviewer expecting the usual
"beta means enabled" ladder should read that difference first, because it is what makes the risk profile
of this feature unlike most.

**The one hard sequencing rule.** Decompression is unconditional and a written frame is a permanent,
one-way commitment, so ordering protects downgrade rather than upgrade:

- No release may make frame-writing reachable by default, or by any configuration action short of
  enabling an off-by-default gate, unless the previous minor release can already read a frame.
- A release that makes frame-writing reachable only from behind an off-by-default gate must state, in
  the release note and in the flag help, that opting in forfeits downgrade to the previous minor for the
  selected resources until a migration has rewritten them uncompressed.

v1.38 satisfies the first rule only through the gate, since nothing can teach v1.37 to inflate. That is
why the write path needs two independent acts — an off-by-default gate *and* an authored file — and why
the second rule applies to it. Two consequences follow: **beta cannot be pulled forward into v1.38**,
and **any new algorithm must ship read support one release before any release may write it.**

#### Alpha

Targeted at v1.38.

- [ ] The gate is alpha and off by default; writes are reachable only with a configuration file and are
  disabled by removing it; a malformed or contradictory configuration fails startup rather than silently
  doing nothing.
- [ ] **Open question 1 decided: whether a frame carries a checksum.** This has to be settled before
  alpha ships rather than at beta, because adding integrity afterwards requires a new algorithm id and a
  release of read-before-write skew, while deciding it now costs four bytes and roughly one percent of
  compression CPU. The measurements are in [Integrity](#integrity-what-authenticates-a-frame). If the
  answer is a checksum, a mismatch needs a classification distinct from a format error so that it stays
  eligible for the unsafe-deletion flow of KEP-3926.
- [ ] Five metrics registered at ALPHA stability, matching the metrics list in `kep.yaml`.
- [ ] The release note and the flag help carry the downgrade-forfeit warning above. If open question 1
  is decided against a checksum, they also carry the caveat that a frame's only integrity protection is
  the encryption provider's — which means none at all under AES-CBC or with encryption disabled.
- [ ] The unit and integration coverage described in the [Test Plan](#test-plan). The integration tests
  do not exist today.
- [ ] Benchmarks establishing write-latency and CPU cost, published as the baseline against which beta is
  measured rather than to meet a threshold. Two artifacts gate alpha rather than merely reporting: B1's
  inert-path allocation count, and tier B5's safety bounds, which are correctness properties with numbers
  attached.
- [ ] **A sig-auth review of the compressibility side channel specifically**, not of the KEP generally.
  The residual and its threat model are in [Risks and Mitigations](#risks-and-mitigations). The review
  decides whether that residual is acceptable at alpha, whether per-resource opt-in and the absence of
  any ratio or byte-count metric are jointly sufficient, and whether the startup warning for an
  explicitly named `secrets` is adequate. Sign-off is recorded in Implementation History.

Open question 2, a per-resource compression level, is a beta item. Open question 3, the discriminator
invariant belonging in apimachinery, is carried by beta's closing of known gaps.

#### Beta

Targeted at v1.39.

- [ ] The gate becomes beta and on by default, but the write path still requires a configuration file, so
  the effective default does not change. Writes remain disableable by removing the flag; reads are never
  disableable.
- [ ] **Open question 2 resolved**: whether the compression level becomes per-resource. It needs no
  format change, since the level is a writer-side choice the decoder never reads, so it can be added
  here on the evidence in the level appendix — level 4 buys 22.7% more ratio for 1.38x the CPU on a
  170 KB Pod, while level 9 is dominated everywhere.
- [ ] **The configuration shape is finalised** — either promoted to a beta API version or kept at
  v1alpha1 with a stated reason — along with the size floor, the size ceiling and the inflation
  concurrency limit.
- [ ] **Raw-etcd tooling can read a frame.** This is the one dependency outside kubernetes/kubernetes:
  `auger`, `etcdhelper` and snapshot forensics parse etcd values directly, and in an unencrypted cluster
  the stored value *is* the frame, so today those tools simply fail on one. Beta needs a released `auger`
  that recognises the header and inflates the body, plus a documentation page stating which tools read
  the frame at which versions.
- [ ] **A documented and tested downgrade procedure**, exercised in the migration integration suite:
  deselect the resource, restart the control plane, run a migration per affected resource, wait for it to
  succeed, and only then downgrade — asserting afterwards that no stored value for that resource begins
  with the frame discriminator.
- [ ] **Performance evidence on both sides.** With an empty policy, a large-scale performance run inside
  the existing p99 SLO thresholds, demonstrating that an inert installation costs nothing. With a policy
  on a large resource, the full benchmark tiers no worse than the alpha baseline, plus measured peak
  writes per second and extra revisions during the one-time amplification burst so that an operator can
  size it.
- [ ] **Monitoring complete and interpretable**: a quiet rewrite counter means converged, a non-zero
  unsupported-algorithm counter means skew or tampering with peer versions to be correlated first, and a
  non-zero format-error counter means investigate rather than roll back. No ratio or byte-count metric
  added in the meantime.
- [ ] **Feedback from alpha adopters** on which resources they selected and why, how they drove the
  amplification burst, whether the raw-etcd tooling gap blocked an investigation, and whether "no metric
  proves absence" was understood before or only after a downgrade attempt. Known gaps closed, including
  the apimachinery note of open question 3.

#### Deprecation

**The read side can never be deprecated.** Once any cluster has written a frame, every subsequent
kube-apiserver must be able to inflate one, indefinitely. The algorithm ids carry no sunset date,
because notice is useless against data already at rest.

**"Removal" can therefore only mean removing the write path** — the flag, the configuration type, the
policy and the writer — and even that requires a migration first. Removing the writer removes the
policy, which makes every framed value stale at once and rewrites it uncompressed on its next update:
terminating, but an uncontrolled amplification burst on precisely the largest resources. The sequence
would be to announce; keep the write path, including the ability to select the `None` algorithm, for at
least two releases and at least the standard deprecation window for the configuration type's stability
level; document migration to uncompressed as a prerequisite; and only then remove the writer.

**Nothing existing is deprecated by this KEP.** `--storage-compression-config` is new and supersedes
nothing — not `--storage-media-type`, not response compression, not any etcd-side setting — so the
template's item about announcing deprecation of an existing flag has no subject. The gate itself
deprecates ordinarily, with the flag and configuration type outliving it, as other configuration files
have outlived their gates.

### Upgrade / Downgrade Strategy

**Upgrade is a no-op.** The feature gate is off by default and `--storage-compression-config` is unset,
so an untouched cluster stores byte-identical values after the upgrade. The decompression path is
present regardless of the gate but stays inert: it finds no frames, records no metrics, and returns
every value unchanged. There is no ordering requirement among apiservers, because a read is decided by
the stored bytes rather than by the reading apiserver's configuration.

**Enabling a resource costs a one-time write amplification.** Once a resource is selected, each of its
objects between the size floor and the ceiling is reported stale on read, which deliberately defeats
the optimisation that normally skips an etcd write when a client's update serialises to bytes identical
to what was read. Idempotent controller updates that previously cost nothing each become one real write
and one revision, until every object has been rewritten once. It is bounded and self-terminating, but
on a large cluster with chatty controllers it is a visible burst in writes, revisions and
pre-compaction database growth. Driving it deliberately with a storage version migration (KEP-4192,
whose controller has been enabled by default since v1.37) is preferable to waiting for organic churn.
Where that controller is disabled the burst still terminates, but its timing is not under the
operator's control.

**Disabling is safe for reads at any time and can never strand data, but it decompresses nothing.** An
operator can remove the resource from the configuration, exclude it ahead of a wildcard by giving it
the `None` algorithm, or drop the flag and the gate together. Existing frames stay on disk until
something rewrites them: the staleness verdict flips direction and they migrate back by the same
mechanism, at the same amplification. Neither disabling nor running with an emulated version below the
first supporting release sheds a frame — the write path becomes unreachable while the read path stays
active.

**Graduating from alpha to beta changes a default, not a behaviour.** At beta the gate defaults to on,
but framing still requires a configuration file that names a resource, so a cluster without one sees no
change and the effective default remains "compress nothing". Compression also cannot switch itself on
for a cluster that does have a file: because setting `--storage-compression-config` without the gate
fails startup at alpha, no alpha cluster can be running with a file present and the gate off, so there
is no staged configuration waiting to activate itself on upgrade. That startup failure earns its keep
here. From beta to GA the gate is locked on and nothing further changes, the configuration file
remaining the only switch.

**Downgrading from beta back to alpha is a configuration hazard rather than a data one.** Frames written
at beta stay readable at alpha, so no data is at risk. But a beta cluster that relies on the gate's
default and has a configuration file will meet the alpha default of off with the flag still set, and
refuse to start. Before crossing that boundary downwards, either set the gate explicitly to on or
remove the configuration file. This is the one transition where the safety check is itself the
obstacle, and it fails loudly at startup rather than quietly at runtime.

**Downgrade below the first decompress-capable release is the serious case, and it is an outage rather
than a degradation.** Decryption succeeds, because the encryption envelope is unchanged and in an
unencrypted cluster the frame simply *is* the value; the frame then reaches a storage codec that
recognises nothing. A single undecodable value removes serving for its entire resource, indefinitely:

- A LIST fails entirely rather than partially, because the older apiserver aborts on the first value it
  cannot decode instead of aggregating per item.
- A WATCH terminates on the first undecodable event, so the watch cache loses its reflector and answers
  every subsequent re-list with an initialisation error.
- The affected objects are classified as corrupt, because the older binary lacks the exclusion that
  keeps a format error out of that category. That makes undamaged objects eligible for unsafe deletion.
  The correct response is to roll forward, not to delete.

**The only sound preparation is a completed migration per compressed resource**, in this order: remove
those resources from the configuration on *every* apiserver and restart, then run a migration for each
and wait for it to succeed, and only then roll back. The order matters, because a migration run while
the resource is still selected rewrites each object into a fresh frame and achieves nothing.

**What to monitor**, beyond the metrics in [Observability](#observability):
`etcd_requests_total{operation="update"}` for the amplification, etcd revision and database-growth
signals alongside it, and `apiserver_storage_size_bytes` for the objective. After a downgrade, on the
older binary, `apiserver_storage_decode_errors_total` and the watch-cache initialisation counters are
where this failure appears.

### Version Skew Strategy

This is a kube-apiserver-only change to the at-rest storage format. It adds no REST API type, no served
field and no negotiated wire capability, so kubelet, kube-proxy, kube-scheduler,
kube-controller-manager, CSI/CRI/CNI, `kubectl` and client-go carry no new constraint, and etcd sees an
opaque value either way. The only skew that matters is between apiservers backed by the same etcd, plus the
configuration file itself, which no earlier release can parse.

**Apiserver N/N-1.** An apiserver from the previous release has no decompression path, so a frame
reaches its decoder and fails, taking the whole LIST with it. Hence the governing rule: **no apiserver
may write a frame until every apiserver backed by the same etcd can read one.** Because read and write
support ship together in the same release, that rule is discharged by operator sequencing rather than by
a release boundary — upgrade every apiserver, confirm the rollout, then add the configuration file.
Writing requires two deliberate, restart-scoped acts: enabling `StorageObjectCompression`, and pointing
`--storage-compression-config` at a file that names a resource with an algorithm other than `None`.
Setting the flag without the gate fails startup rather than silently doing nothing. Reverting either act
stops new frames from being written but leaves every existing frame readable, because the read path
never consults the policy.

**Mixed configuration across apiservers is the case an operator can actually create.** Reads are
unaffected: the read path is installed unconditionally and decides from the stored bytes, so a frame
written by one apiserver is readable by every other whatever their policies say. Writes are a different
matter. The same predicate decides both whether to frame a value and whether a stored value is stale,
so two apiservers with different configurations disagree about staleness on exactly the objects they
disagree about writing. A mid-sized ConfigMap is then rewritten framed by one and bare by the other,
indefinitely. Each rewrite converges for the apiserver that performed it, but the fleet as a whole does
not, so the one-time amplification burst becomes permanent. There is no data loss and no correctness
problem — only sustained and pointless write traffic.

**The guidance is therefore to keep the configuration identical on every apiserver backed by the same
etcd**, exactly as the encryption configuration already requires. The file is read once per process,
so disagreement comes from a partial rollout, or from an in-place edit that only a later restart applies
to one replica. No automated detection is proposed; the symptom is
`apiserver_storage_compression_rewrites_needed_total` climbing in both directions at once and never
going quiet.

An unsupported algorithm cannot be reached within any supported skew at alpha, since only two
algorithms exist and both are readable from the first supporting release. Any future algorithm must
ship read support one release before any release may write it. Why that error is classified as skew
rather than corruption, and why GOAWAY was rejected as a response to it, are in
[Design Details](#error-classification-and-version-skew).

Custom resources and aggregated apiservers are covered rather than excluded: the configuration reaches
every store, including the one built for custom resources, and each aggregated apiserver stores its own
resources under its own keys, with its own configuration.

### Open Questions

**1. Should the frame carry a checksum, or should compression be documented as requiring an AEAD
provider?** The measurement in [Integrity](#integrity-what-authenticates-a-frame) shows that for
`aescbc` and for unencrypted clusters, compressing raises single-bit-flip detection from 4.50% to
72.08% but raises expected silently-corrupted bytes per flip from 0.955 to 51.6. Four options, offered
for reviewer input rather than decided here:

- **CRC32 of the plaintext in the frame.** +4 bytes, taking framing overhead to 9 bytes for `stored` and
  10-12 for `deflate`, at most 1.17% at the size floor, and
  roughly 1% of compression CPU: CRC32 measures 25.0 GB/s (Castagnoli) and 45.4 GB/s (IEEE) against
  DEFLATE level 1 at 254.7 MB/s on the same payload and machine. Detection goes to ~100%. This needs a
  distinct sentinel *excluded* from the recoverable-format classification, because a malformed frame is
  our bug whereas a checksum mismatch is real corruption and *should* be eligible for the
  unsafe-deletion flow of KEP-3926.
- **Use gzip rather than raw DEFLATE.** Gets a CRC32 for +18 bytes with no hand-rolled integrity code —
  larger, but a reasonable answer if reviewers would rather not review a custom frame.
- **Document that compression should be paired with an AEAD provider.** Zero cost, but it protects
  nobody who ignores it, and since encryption at rest is opt-in this leaves the exposure in place for
  what is likely the majority of clusters.
- **Make the checksum configurable.** Not recommended: it doubles the format matrix and makes a frame's
  meaning depend on configuration.

**2. Should the DEFLATE level become per-resource at beta?** Alpha fixes level 1. The measurements in
[the level appendix](#appendix-measuring-the-compression-level-trade-off) do not support fixing it
forever: below ~34 KB the level is nearly irrelevant (3–7% ratio for 1.3–1.7x CPU), but at 170 KB
level 4 buys +22.7% ratio for 1.38x CPU. Level 9 is dominated everywhere and should be documented as
such. A per-resource level is a plausible beta addition for large-object resources; it is deliberately
out of scope for alpha, since the level is a writer-side choice the decoder never needs and can
therefore be added without a format change.

**3. The `0x00` discriminator is an invariant about apimachinery that this KEP does not own.** It
deserves a note in apimachinery rather than living only here and in one test.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `StorageObjectCompression`
  - Components depending on the feature gate: kube-apiserver
- [x] Other
  - Describe the mechanism: both boxes are checked because neither act suffices alone. Framing needs
    the gate **and** `--storage-compression-config` naming the resource with an algorithm other than
    `None` in a `StorageCompressionConfiguration`. The gate alone is inert at every stage including
    GA; the flag alone fails startup.
  - Will enabling / disabling the feature require downtime of the control plane? No, though every
    kube-apiserver restarts in either direction, the configuration being read once per process with
    no hot reload; on an HA control plane that is a rolling restart.
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? No. No node
    component participates, and a node's view of the API is unchanged.

###### Does enabling the feature change any default behavior?

No. Without a configuration file the gate leaves stored bytes byte-identical. Selecting a resource
changes only the bytes at rest for it, plus a one-time write amplification ([Upgrade / Downgrade
Strategy](#upgrade--downgrade-strategy)). Nothing visible through the API changes, but raw-etcd
tooling can no longer read those values; see [Risks and Mitigations](#risks-and-mitigations).

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

The write path, yes, immediately and with no data at risk: remove the resource's entry, or give it
the `None` algorithm, with no gate change. A rollback expressed through the gate must drop the flag
in the same change, since the flag without the gate fails startup. The read path, no — decompression
is unconditional and cannot be turned off, so rollback can never strand data.

Consequently **rollback does not mean uncompress**: existing frames stay readable indefinitely and
revert to bare values only as something rewrites them, at the same amplification cost. Reverting the
*bytes* — needed only when downgrading below the first decompress-capable release — takes a storage
version migration per resource, and no counter can confirm that no frames remain.

###### What happens if we reenable the feature if it was previously rolled back?

Exactly as first enablement: one restart and one more bounded amplification burst over what was
rewritten bare in the interim. Values still framed are not rewritten, so nothing accumulates across
cycles, and a half-migrated resource is a steady state that every configuration reads correctly.

###### Are there any tests for feature enablement/disablement?

Yes. Unit tests assert that an unconfigured apiserver frames nothing and emits no metric samples at
all rather than a row of zeros; that the flag without the gate is rejected; that a stale value
converges in exactly one rewrite in either direction, the property making enable and disable
terminate; and that a value framed under one configuration reads back under any other, including
none. The restart-driven cycle and the inertness and startup-failure cases are alpha deliverables
([Test Plan](#test-plan)).

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

No workload is affected: only how kube-apiserver stores bytes changes, and etcd sees an opaque value
either way. Nor can any interleaving corrupt data, because a read is decided by the stored bytes and
never by the reading apiserver's configuration — see [The read path](#the-read-path) — so a mid-rollout
fleet needs no coordination, and an apiserver that restarts, or never receives the file, still reads
what its peers wrote.

Every remaining failure mode lands on the control plane. **A configuration mistake fails startup rather
than degrading**: a malformed file, an unknown algorithm, an unreadable path, a resource selector
already claimed in full by an earlier entry — which fails the whole file even when other selectors in the
same entry are live — or `--storage-compression-config` without the `StorageObjectCompression` gate each
refuse to bring that apiserver up, so a rolling update loses one replica at a time while the rest keep
serving and the operator gets a crash loop with an actionable message rather than silent inaction.
Applying an unvalidated file to every replica at once is what makes that an outage. **Inconsistent
configuration across apiservers** — a partial rollout, or an in-place edit only some replicas have
restarted into — is the other operator-caused failure, and its symptom is sustained rewrite churn rather
than a data problem: see [Version Skew Strategy](#version-skew-strategy).

Rollback within the same minor is a configuration change applied at the next restart: new frames stop
and existing ones stay readable until something rewrites them. Rollback below the first
decompress-capable release is the one genuinely dangerous transition and is an outage — see
[Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

###### What specific metrics should inform a rollback?

One signal argues for switching the feature off, none argues for deleting an object, and none can clear
a downgrade, since no counter can prove the absence of frames.

- Sustained p99 write-latency or apiserver CPU regression on a selected resource, or etcd write and
  revision rate still elevated beyond what the one-time burst accounts for: remove that resource from
  the configuration. This is the rollback signal.
- `apiserver_storage_compression_rewrites_needed_total` climbing in both directions and never going
  quiet: inconsistent configuration; reconcile the file, because rolling back does not fix it. Climbing
  in one direction without decaying: the migration is not converging — investigate first.
- `apiserver_storage_compression_unsupported_algorithm_total` non-zero: a peer writes frames the reader
  cannot inflate. Roll the lagging apiserver *forward*.
- `apiserver_storage_compression_format_errors_total` non-zero: investigate, and do not roll back. An
  older binary reads the same bytes less well, not better.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

No, and the upgrade → downgrade → upgrade path has not been run either. What exists is unit-level and
manual: golden at-rest bytes, round trips in both directions, staleness converging in exactly one
rewrite, framing verified inside real ciphertext, inertness with no configuration present, and fuzzing
of the read path against arbitrary bytes. Those establish that both directions of the format transition
are correct at the storage layer and say nothing about a control plane surviving them; the measurements
reported elsewhere here were taken by hand, not in CI. Integration tests do not exist at all today,
their shape being an alpha deliverable in the [Test Plan](#test-plan); the two transitions most worth
running are the beta-default-on to alpha-default-off boundary, where the startup check is itself the
obstacle, and the pre-read-path downgrade preceded by a completed migration, a beta graduation item.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No. `--storage-compression-config` is new and supersedes nothing, and no existing flag, field, API or
behaviour is deprecated or removed. What removal could mean later, and why the read side can never be
one, is in [Deprecation](#deprecation).

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Not by workloads — none can enable or observe this. The operator inputs decide: the gate
`StorageObjectCompression` plus a `StorageCompressionConfiguration` at `--storage-compression-config`.
Absent either, nothing is framed. `apiserver_storage_compression_operations_total` then reports per
resource, its `outcome` label separating compressed from framed-but-verbatim from unframed values.

###### How can someone using this feature know that it is working for their instance?

- [ ] Events
- [ ] API .status
- [x] Other (treat as last resort)
  - Details: deliberately no end-user signal; end users cannot read `/metrics`. The operator watches the
    compressed `outcome` on `apiserver_storage_compression_operations_total` per selected resource for
    confirmation that framing is happening, and `apiserver_storage_size_bytes`, which is per etcd cluster
    rather than per resource, for the benefit.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Both error counters — `apiserver_storage_compression_format_errors_total` and
`apiserver_storage_compression_unsupported_algorithm_total` — at zero always; non-zero means
investigate, not roll back. `apiserver_storage_compression_rewrites_needed_total` quiet again within
one migration per resource added or removed, and `apiserver_storage_compression_duration_seconds` p99
well inside the existing request-latency SLOs, which remain the real ceiling.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: `apiserver_storage_compression_operations_total`,
    `apiserver_storage_compression_duration_seconds`,
    `apiserver_storage_compression_format_errors_total`,
    `apiserver_storage_compression_unsupported_algorithm_total`,
    `apiserver_storage_compression_rewrites_needed_total`
  - [Optional] Aggregation method: rate by resource and outcome; p99 by operation; any non-zero total
    for either error counter; rate by resource and direction, watched for reaching and holding zero.
  - Components exposing the metric: kube-apiserver, and aggregated apiservers using the same layer.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No compression-ratio or byte-count metric, and no label naming *why* a frame was rejected — a
content-dependent signal and a format oracle respectively (see [Observability](#observability)); the
ratio question is answered offline by the snapshot analyzer in the [Test Plan](#test-plan). Nothing else
could be added safely: the measurement an operator most wants, how much a given resource actually saves,
is precisely the content-dependent signal excluded above. A count of frames remaining can never be a
metric either, since the counters are read-driven and an untouched object is never observed — hence the
migration-based pre-downgrade procedure in [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. Nothing new runs in the cluster: no service, no node-level agent, no controller, no external or cloud
API, no extra binary and no new third-party dependency. Both paths run in kube-apiserver itself, alongside
the serialisation and encryption already applied to every stored object. Of the three items below, only
etcd is a runtime dependency, and none of them can stop a stored value from being read.

- **etcd**
  - Usage description: unchanged. It stores an opaque value either way, and no particular etcd version,
    setting or feature is required.
    - Impact of its outage on the feature: identical to its impact on storage today.
    - Impact of its degraded performance or high-error rates on the feature: none beyond the existing
      effect on every read and write.
- **Storage version migration (KEP-4192)**
  - Usage description: not needed to serve reads or writes. It is how an operator schedules the one-time
    write amplification, and a completed migration per compressed resource is the only sound preparation
    for a downgrade below the first decompress-capable release; see
    [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).
    - Impact of its outage on the feature: compression keeps working and the amplification still
      terminates, but on organic churn rather than on the operator's schedule, and no sound downgrade
      preparation is available until it returns.
    - Impact of its degraded performance or high-error rates on the feature: a slow or partial migration
      only lengthens the amplification window.
- **Raw-etcd tooling (auger, etcdhelper)**
  - Usage description: unused by the feature; used by operators to read stored values directly, and
    unable to parse a frame today.
    - Impact of its outage on the feature: none on serving. It costs forensics on an unencrypted cluster,
      where the stored value is the frame; frame-aware tooling is a beta item, see
      [Graduation Criteria](#graduation-criteria).
    - Impact of its degraded performance or high-error rates on the feature: none.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No new call, and nothing lists, watches or reconciles what it did not before. One class of existing write
newly reaches etcd: while a selected resource migrates, an update serialising to bytes identical to what
was read is no longer skipped; see [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

###### Will enabling / using this feature result in introducing new API types?

No. StorageCompressionConfiguration is a configuration-file kind read once at kube-apiserver startup,
never persisted and never served, so no object limit applies to it.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No. Compression sits beneath encryption, so a KMS-backed cluster makes the same number of provider calls
at the same points; only the size of the plaintext handed to the provider changes.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Object count is unchanged, served objects are byte-identical, and no annotation, label or field is added.
Stored size decreases; ratios are in [Measured on production data](#measured-on-production-data). Only a
selected value above the 1 KiB floor that does not compress grows, and then by exactly the five-byte
verbatim-frame overhead: 0.49% at the floor, less above it.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

With no configuration file nothing changes anywhere. For a selected resource each mutating request adds
one compression pass and each read of a framed value one inflation, cheaper by 1.7-2.0x; see
[the level appendix](#appendix-measuring-the-compression-level-trade-off). The worst read case is
watch-cache initialisation, a full LIST inflated object by object while the request waits. Stored size is
not an SLI, so the benefit is invisible to the SLO framework and the cost is not; beta measures both.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Only kube-apiserver: CPU on the write path for selected resources, on the read path for framed values.
Memory rises in the pooled compressors (12-18 MiB at a parallelism of 16) and in concurrent inflation (at
worst the concurrency cap times roughly 1.54 MiB); see
[the memory appendix](#appendix-measuring-compressor-and-decompressor-memory-footprint). Disk and IO fall
in steady state, after a temporary rise during the one-time burst. Admission control must also be
corrected, since it charges a LIST by stored bytes while the memory it occupies is the plaintext; see
[API Priority and Fairness](#api-priority-and-fairness-and-size-accounting).

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Nodes are unaffected: this is a control-plane change, and no PIDs, sockets, inodes or descriptors are new.
Control-plane memory is exhaustible, through inflation of bytes an actor with etcd write access chooses,
since two supported configurations authenticate nothing; it is bounded explicitly in declared size and in
concurrency, see [Bounding decompression](#bounding-decompression). Those bounds and the admission ceiling
are validated in the adversarial benchmark tier, alongside a mandatory fuzz target; see
[Test Plan](#test-plan).

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

Synchronous, in-process and stateless between requests, it meets etcd unavailability as any other storage
operation does: nothing to drain, reconcile or recover, and nothing runs with the API server down.

###### What are other known failure modes?

- **A malformed frame** — stored bytes look framed but do not decode.
  - Detection: `apiserver_storage_compression_format_errors_total` non-zero; the whole LIST fails.
  - Mitigations: none in place. Under an authenticated provider this is a defect in this layer; under
    `aescbc` or unencrypted the bytes are damaged, so restore from backup — unsafe deletion is refused.
  - Diagnostics: which check rejected the frame is logged at v4, deliberately not a metric label.
  - Testing: read-path fuzzing today; aimed corruption under an unauthenticated provider at alpha.
- **An unsupported algorithm** — a newer API server wrote the frame.
  - Detection: `apiserver_storage_compression_unsupported_algorithm_total`, by resource.
  - Mitigations: upgrade the lagging API server; correlate peer versions first. Not a rollback or a delete.
  - Diagnostics: the unrecognised algorithm id and storage key are logged, naming object and writer.
  - Testing: an alpha integration test plants an unimplemented algorithm; unreachable in a supported skew.
- **Reads queueing on the inflation limit** (see [Bounding decompression](#bounding-decompression)).
  - Detection: read latency on a compressed resource rises while
    `apiserver_storage_compression_duration_seconds` does not — the cost is the wait, not the work.
  - Mitigations: shed concurrent etcd-served LISTs through APF. The limit is derived today; making it
    configurable is proposed in [Bounding decompression](#bounding-decompression).
  - Diagnostics: an abandoned read stops waiting, surfacing as a cancellation rather than as latency.
  - Testing: the adversarial benchmark tier drives concurrent LISTs to the admission limit.

###### What steps should be taken if SLOs are not being met to determine the problem?

Rule the layer out first: a silent `apiserver_storage_compression_operations_total` means this API server
does nothing here, and is no evidence about what etcd holds. Write latency: compare a selected resource
with an unselected one of similar size; the remedy is deselection, effective at restart. Write volume:
growth that does not decay means an unfinished migration or disagreeing configurations, see [Version Skew
Strategy](#version-skew-strategy). Read latency: the wait above. Stored size flat: the `outcome` label
separates unframed values — size floor or policy — from values stored verbatim.

## Implementation History

- 2026-08: A design proposal covering the ordering constraint, the at-rest frame, the write and read
  paths, the decompression bounds and the compressibility side channel circulated ahead of the KEP.
- 2026-08: KEP-6264 opened as provisional, owned by sig-api-machinery with sig-auth, sig-etcd,
  sig-scalability and sig-instrumentation participating; alpha v1.38, beta v1.39, GA v1.41.
- 2026-08: A working implementation exists behind StorageObjectCompression — the frame, the
  per-resource policy loaded from `--storage-compression-config`, unconditional decompression with
  bounded inflation, the five ALPHA metrics, the API Priority and Fairness expansion correction, and
  the skew and format-error classifications — with unit tests and read-path fuzzing. Integration
  tests and the offline snapshot analyzer are not written; see [Test Plan](#test-plan).
- Still to be recorded here: the sig-auth review of the side channel, the decision on Open
  Question 1, and the SIG Architecture and SIG Testing determinations on the release-signoff items
  this KEP reports as having no applicable content. See [Graduation Criteria](#graduation-criteria).

## Drawbacks

- **The at-rest format changes one way.** Once any cluster writes a frame, every future
  kube-apiserver must be able to inflate one, forever; there is no deprecation path for the read
  side, and no notice period helps data already at rest. That is a permanent maintenance obligation
  for a benefit that is opt-in at every stage, GA included, and that most clusters never take.
- **Clusters that never enable writing still carry the read path.** It is installed unconditionally,
  must stay correct against hostile bytes under the unauthenticated providers, and must be reviewed
  and fuzzed on every change. The ratio of permanent surface to realised users is defensibly bad.
- **It breaks raw-etcd tooling until that tooling is updated**, and the fix is outside
  kubernetes/kubernetes; see [Risks and Mitigations](#risks-and-mitigations). Until a tool release
  lands, an operator who opts in gives up an incident-time capability.
- **It converts a length leak into a compressibility leak.** Narrow but real, and mitigated by policy
  and defaults rather than by mechanism, so the residual is accepted rather than closed. A project
  unwilling to add any content-dependent channel at all can coherently reject this outright.
- **It spends CPU on the request goroutine** for every read and write of a selected resource,
  including watch-cache initialisation; KEP-2338's regression is the cautionary precedent. Enabling a
  resource also costs a one-time write-amplification burst by design, and an incompressible object
  above the size floor grows by the frame header, permanently.

## Alternatives

- **Compress anywhere but strictly inside the transformer chain** — in etcd, in its storage engine,
  in the filesystem, or above the chain. All yield a ratio near 1.0 under encryption at rest, the
  motivating case, because everything outside the chain sees only ciphertext.
- **A new on-disk prefix rather than a frame.** Rejected; see
  [Notes/Constraints/Caveats](#notesconstraintscaveats).
- **gzip instead of raw DEFLATE.** Same algorithm, +18 bytes per value, and a checksum without custom
  integrity code; still a live option under Open Question 1. Raw DEFLATE is the alpha choice because
  the frame already needs a discriminator and a declared length, so gzip's header duplicates that
  work and only its trailer adds anything. See [Integrity](#integrity-what-authenticates-a-frame).
- **A shared compression dictionary across objects.** The obvious way to help objects near the size
  floor, and permanently rejected: shared state across values creates exactly the cross-object oracle
  that [Risks and Mitigations](#risks-and-mitigations) argues is structurally impossible today, which
  is why each value is compressed with a freshly reset compressor.
- **A write-path dry-run mode that compresses and discards to estimate benefit.** It samples the
  write stream rather than the stored set, answering a different question than "what would my
  database shrink to"; it pays full compression CPU on the hot path for no stored saving; and its
  per-resource realised ratio is exactly the finer content-dependent signal
  [Observability](#observability) refuses to export. An offline analyzer answers the question better and
  in private. Were a live estimate ever wanted, sampling on the *read* path would be the least-bad
  shape: it observes the stored set rather than the write stream, cannot affect what is written, and
  needs no second framing predicate.

## Appendix: measuring the compression level trade-off

**What was measured.** Compression CPU and achieved ratio for DEFLATE levels 1, 2, 3, 4, 6 and 9;
decompression CPU, which is level-independent; and retained heap per compressor and decompressor, whose
method is documented in [the memory
appendix](#appendix-measuring-compressor-and-decompressor-memory-footprint).

**Fixtures.** Five real Pods drawn from the production audit-log samples in the motivation section, one
from each of five of the seven size classes there, converted to the **protobuf storage form** — the actual at-rest plaintext, not
the JSON the audit log recorded. Resulting plaintext sizes: 3.5 KB, 5.8 KB, 33.8 KB, 170 KB, 287 KB.
Ratios therefore differ across rows because of object *content*, not size alone; each row is a single
object, so the rows are not a size sweep of one object.

**CPU method.** `go test -bench`, 200–300 iterations per case. A single `flate.Writer` is allocated per
case and reused across iterations via `Reset`, matching the production write path, which pools writers.
Without that reuse, allocating a fresh writer costs 0.8–1.2 MB and dominates the measurement on small
payloads, inverting the apparent level ordering. Reported allocation is consequently ~34 B/op at 0
allocs/op, so the figures are compression work only. Decompression reuses one `flate.Reader` via
`flate.Resetter`. Encryption, serialization and the etcd round trip are all excluded.

**Results.** Compression time / achieved ratio, with level-independent decompression time in the last
column:

| plaintext | L1 | L2 | L3 | L4 | L6 | L9 | inflate |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 3.5 KB | **38.1 µs / 1.75x** | 68.2 / 1.79 | 54.0 / 1.80 | 55.2 / 1.81 | 56.1 / 1.82 | 58.4 / 1.82 | 18.9 µs |
| 5.8 KB | **45.0 / 2.18** | 62.5 / 2.21 | 63.0 / 2.22 | 72.0 / 2.24 | 71.3 / 2.25 | 73.3 / 2.25 | 24.0 |
| 33.8 KB | **288 / 2.61** | 321 / 2.70 | 341 / 2.72 | 392 / 2.77 | 461 / 2.79 | 481 / 2.79 | 148 |
| 170 KB | **1062 / 4.09** | 1168 / 4.52 | 1282 / 4.62 | 1466 / 5.01 | 2219 / 5.38 | 4720 / 5.41 | 610 |
| 287 KB | **1125 / 6.00** | 1495 / 6.07 | 1827 / 6.11 | 1986 / 6.45 | 4530 / 6.60 | 18999 / 6.84 | 647 |

**Conclusions.**

1. **Below ~34 KB the level barely matters.** Level 1 to level 9 buys 3–7% ratio for 1.3–1.7x CPU.
2. **Above ~170 KB it matters a great deal.** Level 4 buys +22.7% ratio for 1.38x CPU on the 170 KB Pod,
   where level 6 wants 2.09x for +31.7%.
3. **Level 9 is never worth it.** On the 287 KB Pod it costs 16.9x level 1's CPU for +14% ratio, and
   +3.7% over level 6 for 4.2x level 6's CPU. It should be documented as dominated, not merely as "not
   the default".
4. **Decompression is level-independent** at 184–443 MB/s: 1.7–2.0x faster than level-1 compression and
   up to 29x faster than level 9. This is the concrete asymmetry behind read-unconditional,
   write-gated.
5. **Level 1 is the right alpha default** — the lowest and most predictable CPU, and the tail is what
   governs p99 write latency.

**Caveats.** Single machine, one object per size class, no concurrency, and no GC pressure from a live
apiserver. This appendix establishes the *shape* of the trade-off well enough to choose a default; it
does not replace benchmarks B1–B5, which measure end-to-end write latency and apiserver CPU under load.

## Appendix: measuring compressor and decompressor memory footprint

**Why this is measured separately.** Compression CPU is a per-write cost that surfaces in latency
benchmarks. Compressor memory is not: it is *retained* state held by a pooled `flate.Writer` for the
lifetime of the process, so it never appears in an allocation profile of a steady-state apiserver and is
invisible to `B/op` in a benchmark that reuses writers. Two decisions depend on it — the DEFLATE level,
which changes the footprint by roughly 50%, and the concurrent-inflation cap, which
bounds decompressor state and output buffers.

**Method.** Retained heap is measured by constructing *N* live objects, differencing
`runtime.MemStats.HeapAlloc` across an explicit GC on each side, and keeping the objects reachable past
the second sample so the collector cannot reclaim them. Dividing by *N* amortises the harness's own
fixed cost.

```go
// heapFor reports retained heap per object, in bytes, for n live objects
// produced by newObj. Both samples follow an explicit GC with the objects still
// reachable, so this measures state that is held rather than allocation churn.
func heapFor(n int, newObj func() any) uint64 {
    var before, after runtime.MemStats
    runtime.GC()
    runtime.ReadMemStats(&before)

    keep := make([]any, 0, n) // build n objects and retain all of them
    for i := 0; i < n; i++ {
        keep = append(keep, newObj())
    }

    runtime.GC()
    runtime.ReadMemStats(&after)
    runtime.KeepAlive(keep) // must outlive the second sample
    return (after.HeapAlloc - before.HeapAlloc) / uint64(n)
}
```

Each object is exercised once before being retained — write a payload, `Close`, then `Reset` — because
parts of `flate`'s internal state are allocated on first use, so measuring a freshly constructed writer
would understate a pooled one. `N = 256`, and the payload is 46 KiB of repetitive text, enough to force
a full window and at least one complete block. Run with `-count=1` and no `-benchtime`; this is a test
rather than a benchmark, because the quantity of interest is a level rather than a rate.

**Results** (Go 1.26.0, linux/amd64, `N = 256`):

| object | retained heap |
| --- | --- |
| `flate.Writer` level 1 | **1178.2 KiB** |
| `flate.Writer` levels 2, 3, 4, 6, 9 | **794.4 KiB** |
| `flate.Reader`, any level | **39.7 KiB** |

**Where the bytes go.** Reading `compressor.init` and `initDeflate` in `compress/flate` accounts for the
measurement almost exactly:

| | shared | level-specific | predicted | measured |
| --- | --- | --- | --- | --- |
| L1 | `hashHead` 512K + `hashPrev` 128K | window 64K + tokens 256K + `deflateFast.table` 128K + `prev` 64K | 1152 KiB | 1178.2 KiB |
| L2–9 | same 640K | window 64K + tokens 64K | 768 KiB | 794.4 KiB |

The residual is 26 KiB at *both* levels — the `huffmanBitWriter`, the remaining `compressor` fields, and
Go's allocator size classes — and it being a constant rather than a level-dependent gap is what makes
the attribution credible.

The counter-intuitive part is that **level 1 retains more memory than level 9**, for a
standard-library reason rather than an algorithmic one. `hashHead [1<<17]uint32` and
`hashPrev [1<<15]uint32` are *inline arrays* in the shared `compressor` struct, so all 640 KiB is
allocated at every level — including level 1, which dispatches to the separate `deflateFast` encoder and
never touches them. Level 1 then adds its own 128 KiB match table and 64 KiB `prev` buffer, and
allocates a full `maxStoreBlockSize` token buffer (256 KiB) where levels 2–9 allocate
`maxFlateBlockTokens+1` (64 KiB). Net: level 1 spends 384 KiB more per writer to save CPU, and roughly
640 KiB of every writer at that level is state nothing reads. This is an implementation detail of the Go
version in use and should be re-measured if that changes.

**Consequences for this design.** A writer pool sized to `GOMAXPROCS` costs ~18.4 MiB at 16 writers on
level 1 against ~12.4 MiB on level 6 — single-digit-MiB territory either way, and small enough that
memory should not drive the level choice. Decompression has the opposite shape: a `flate.Reader` retains
only 39.7 KiB, but the inflated output it produces is bounded by `MaxPlaintextBytes` at 1.5 MiB, so the
**output buffer dominates reader state by roughly 38:1**. The memory ceiling implied by a
concurrent-inflation cap of `n` is therefore approximately `n × (40 KiB + 1.5 MiB) ≈ n × 1.54 MiB`,
or ~98 MiB at the formula's ceiling of 64 — the cap is `min(max(GOMAXPROCS*2, 4), 64)`, so a 16-CPU
apiserver derives 32 and about 49 MiB — and only if every concurrent inflation is simultaneously a
maximum-size object, a worst case rather than an expectation.
