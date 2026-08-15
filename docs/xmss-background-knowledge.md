# XMSS for Post-Quantum Lean Ethereum

## A technical, practical background for signer-state testing

**Audience:** protocol researchers and client developers working on Lean Consensus, Hive, and
post-quantum signatures.

**Purpose:** explain the XMSS concepts that are most useful for this project, connect the papers
to the current `leanSig` implementation, and turn the cryptographic invariants into observable
black-box tests.

This is a working engineering reference, not a replacement for the security proofs in the
papers or for the XMSS standard. The `leanSig` repository describes itself as a prototype and
not production or audited cryptographic software. Always pin the exact commit when using these
details in a test plan.

---

## 1. The short version

XMSS is a **stateful, hash-based signature scheme**.

1. Key generation creates a large set of one-time signing keys.
2. Their public keys are committed into one Merkle-tree root.
3. A signer chooses one unused leaf for each epoch/slot.
4. The signature proves both:
   - that the one-time key signed the message, and
   - that the one-time public key is included under the advertised Merkle root.
5. That leaf must never be used to sign a different message again.

The most important operational invariant is therefore:

> For one XMSS key and one protocol epoch/slot, a client must produce at most one distinct
> valid proposal signature.

If the same message is requested twice, deterministic signing may return the same signature.
That is an idempotent replay. The dangerous case is two different block roots signed with the
same key and epoch.

This is different from a receiver deciding what to do with two conflicting blocks. The novel
failure surface for this project is the **signing boundary**: can a client accidentally create
the equivocation in the first place?

---

## 2. Why XMSS appears in Lean Ethereum

Ethereum today uses BLS signatures in its consensus machinery. BLS has useful algebraic
properties, including native aggregation, and the signer can use the same private key many
times without maintaining a one-time-key counter.

XMSS is attractive for a post-quantum transition because its security is based primarily on
cryptographic hash functions rather than on the hardness of a classical number-theory problem.
It is comparatively simple to describe and implement, but that simplicity moves an important
responsibility into state management: the signer must remember which one-time leaf has already
been consumed.

The Ethereum-specific complication is aggregation. Hash-based signatures do not naturally
aggregate like BLS signatures. The 2025/055 paper therefore studies a family of hash-based
multi-signature designs in which a succinct proof can attest that several individual hash-based
signatures are valid. This introduces a second engineering constraint: the hash-based scheme
must be explicit and circuit-friendly enough to be verified inside a proof system. See
[Hash-Based Multi-Signatures for Post-Quantum Ethereum (ePrint 2025/055)](https://eprint.iacr.org/2025/055)
and the expanded [IACR article](https://cic.iacr.org/p/2/1/13).

### Slot, epoch, and leaf index

The papers often use **epoch** and **slot** interchangeably when discussing one signing
opportunity. A client implementation must define the mapping precisely. In this document,
“epoch/slot” means the protocol-level signing position that selects one XMSS leaf; “leaf index”
means the position of that leaf in the lifetime Merkle tree. They are related, but they are not
automatically the same integer in every integration:

- `epoch`/`slot` is a consensus identifier in a block proposal request;
- the XMSS leaf index is the position used by the signing key schedule;
- a client may apply an activation offset, a key period, or another mapping before selecting a
  leaf.

The test-driver contract should expose both the protocol epoch/slot and enough key metadata to
know which signing position was exercised. A test must not silently assume that “slot 42” means
“leaf 42” unless the client contract says so.

---

## 3. Core vocabulary

| Term | Meaning | Why it matters for testing |
|---|---|---|
| **XMSS** | eXtended Merkle Signature Scheme: a Merkle tree of one-time signatures | Stateful by construction |
| **Lifetime `L`** | Number of one-time leaves, normally `L = 2^h` | After `L` valid positions, the key is exhausted |
| **Tree height `h`** | Merkle-tree depth; also `log2(L)` | Determines authentication-path length |
| **Epoch/slot** | Protocol signing position | The replay-protection key for a proposal |
| **Leaf** | One one-time public key in the Merkle tree | Must be used at most once for a distinct message |
| **OTS/WOTS** | One-time signature / Winternitz one-time signature | The per-leaf signature component |
| **Chain** | Repeated application of a tweakable hash | Reveals only a prefix of a chain; verification completes it |
| **Base / Winternitz parameter `w`** | Chain digit range is `0 ... 2^w - 1` | Trades signature size against chain work |
| **Dimension `v`** | Number of chain coordinates in the codeword | Roughly controls the number of chain values in a signature |
| **Codeword `x`** | Vector of chain positions, `x ∈ {0,...,2^w−1}^v` | Message encoding chooses it |
| **`rho`** | Encoding randomness | Makes the encoding randomized and is carried in the signature |
| **Authentication path** | Sibling nodes from a leaf to the Merkle root | Lets a verifier recompute the committed root |
| **PRF** | Pseudorandom function used to derive per-epoch material | Avoids storing every leaf secret explicitly |
| **Tweak** | Domain-separation input to a hash call | Separates chains, epochs, tree levels, and positions |
| **Parameter** | Public hash/encoding parameter | Included in the public key and signature domain |
| **Activation interval** | Epoch range for which a key is valid | Outside it, signing must not proceed |
| **Prepared interval** | Subrange whose bottom Merkle trees are materialized | A `leanSig` operational optimization |
| **Encoding retry** | Re-run encoding with new `rho` when the codeword is invalid | Expected; not the same as key exhaustion |
| **Exhaustion** | No unused signing position remains | Must be a controlled refusal, not a second signature or a crash |

---

## 4. XMSS from first principles

### 4.1 One-time signatures are hash chains

For one message, a WOTS-like signer has `v` independent chains. Each chain has a secret start
value `s_i`. The public endpoint is obtained by applying a tweakable hash repeatedly:

```text
s_i --H--> --H--> ... --H--> public_endpoint_i
        x_i steps revealed by the signer
```

The message is encoded as a vector `x = (x_1, ..., x_v)`. The signature reveals the value after
`x_i` steps in each chain. The verifier applies the remaining `(2^w - 1) - x_i` steps and checks
that it reaches the public endpoint.

The key reason the encoding needs a restriction is that an attacker should not be able to take a
valid signature and continue every chain to obtain a signature for another message. The usual
Winternitz checksum and the generalized constructions in 2025/055 enforce an
**incomparability** property: for two distinct valid codewords, neither vector is coordinate-wise
greater than or equal to the other.

### 4.2 Merkle commitment

Each one-time public key is a leaf. Hashing pairs of leaves repeatedly produces a single root:

```text
             Merkle root
              /       \\
          ...           ...
          / \\           / \\
       leaf  leaf    leaf  leaf
```

The long-lived XMSS public key is essentially this root plus the public parameters. A signature
contains a one-time signature and the sibling nodes needed to open its leaf to the root.

### 4.3 Key generation

At a high level:

1. Choose public hash/encoding parameters `P`.
2. Generate a PRF key.
3. For every epoch and chain index, derive a secret chain start from the PRF key.
4. Walk each chain to its endpoint to form the one-time public key.
5. Build the Merkle tree and publish its root.

The PRF means a client does not need to keep all one-time secret values in memory. It can derive a
chain start from `(PRF key, epoch, chain index)` when needed. It still must protect the logical
state saying which epoch/leaf has been consumed.

### 4.4 Signing

For message `m` at epoch `e`:

1. Check that `e` is inside the activation and prepared ranges.
2. Derive a retry randomness value `rho`.
3. Compute a codeword `x = IncEnc(P, m, rho, e)`.
4. If the encoding rejects, try a new `rho` (up to a configured limit).
5. Derive each chain start using the PRF.
6. Walk chain `i` for `x_i` steps and reveal the resulting value.
7. Attach the Merkle authentication path for the leaf selected by `e`.

The signature therefore carries three conceptually different pieces:

```text
signature = { encoding randomness rho, chain values, Merkle authentication path }
```

### 4.5 Verification

The verifier does not need the private key or the signer’s counter:

1. Recompute the codeword from `P`, `e`, `m`, and `rho`.
2. Reject if the encoding returns `⊥`.
3. For each chain value, apply the remaining hash steps.
4. Reconstruct the one-time public key.
5. Follow the authentication path to recompute the Merkle root.
6. Accept only if the root equals the public key root.

Verification proves that the signature is valid for that message and epoch. It does **not** prove
that the signer has not already produced another valid signature for the same epoch. That is why
the signer’s state must be tested separately.

---

## 5. The encoding layer: where message digits come from

### 5.1 Generalized/incomparable encodings

The generalized XMSS framework in 2025/055 separates the signature scheme from the encoding
choice. The encoding maps `(P, message, randomness, epoch)` to a vector of chain positions. It may
be randomized and may fail with probability `δ`:

```text
IncEnc(P, m, rho, e) = x       or       ⊥
```

The signer retries up to `K` times. If one attempt fails with probability `δ`, the probability
that all `K` attempts fail is approximately `δ^K`; `K` is chosen to make this below the desired
correctness error. A test should therefore distinguish an expected retry from a consumed leaf.

The paper also defines a **target-collision-resistance** property for the encoding. Informally,
given a valid encoding for one message, it should be hard to find a different message and
randomness that produce a second usable encoding for the same signing position. This is a
cryptographic property; it is not the same as a client accidentally calling `sign` twice for an
epoch.

### 5.2 Target-sum Winternitz encoding

The target-sum construction chooses codewords satisfying:

```text
                 sum(x_i) = T
```

The message hash is interpreted as a candidate vector. If its coordinates do not sum to `T`, the
encoding returns `⊥` and the signer retries with another `rho`.

This replaces the ordinary checksum with a fixed total. It has a useful operational consequence:
the verifier performs a predictable amount of chain work. Without a fixed sum, a maliciously
chosen or unlucky encoding could make verification much more expensive than average. The tradeoff
is that a larger target sum generally means more verifier work but fewer signing retries; the
right value is a system parameter.

The 2025/055 analysis recommends choosing `T` near the expected sum of uniformly sampled digits,
with a small upward bias when the system prefers slightly more signing work in exchange for a
lower verification cost. The exact security and correctness bounds depend on the hash security
level, number of queries, dimension, base, and retry count; use the paper’s parameter tables rather
than copying a value into a new implementation.

### 5.3 “Top of the hypercube” encodings (2025/889)

The 2025/889 paper studies the geometry of these codewords. A vector of `v` digits is a vertex of
a `v`-dimensional hypercube. The paper gives a lower bound on verification cost for general
encodings, including randomized, non-uniform, and non-injective encodings, when signature size is
fixed.

Its construction uses a non-uniform map into upper layers of a larger hypercube. The resulting
encoding collisions are designed to be hard to find, while the average verification work improves
by roughly **20–40%** at the same target size and security level in the reported settings. The
idea can be used with XMSS-like hash chains, but it changes the encoding/performance point—not the
rule that one leaf must not sign two different messages.

Read [At the Top of the Hypercube — Better Size-Time Tradeoffs for Hash-Based Signatures
(ePrint 2025/889)](https://eprint.iacr.org/2025/889).

### 5.4 Aborting encodings and the aROM (2026/016)

Some proof-friendly message hashes deliberately reject a small fraction of inputs. The 2026/016
paper models this with an **aborting random oracle**: a query may return “abort” with probability
`θ`, and the proof accounts for those aborts. It gives bounds and indifferentiability results that
connect the model back to the standard random-oracle model, then applies it to SNARK-friendly
incomparable hypercube encodings.

In `leanSig`, the aborting message hash uses rejection sampling over a field:

1. Poseidon produces field elements `A_i`.
2. Values at or above a carefully chosen bound are rejected.
3. Accepted values are decomposed into base-`w` digits.
4. Enough digits are collected to form the `v`-coordinate codeword.

The source comments give the abort probability as approximately:

```text
theta = 1 - ((Q * w^z) / p)^ell
```

where `p` is the field size, `Q` and `z` define the accepted digit range, and `ell` is the
number of Poseidon outputs needed. The KoalaBear example uses
`p = 2^31 - 2^24 + 1 = 127 * 8^8 + 1`, `w = 8`, `z = 8`, and `Q = 127`, giving a very small
per-element rejection probability.

For the Hive project, an abort is an **encoding retry**. It must not be reported as:

- XMSS key exhaustion;
- a duplicate epoch;
- a cryptographic invalid signature; or
- evidence that the client reused a signing leaf.

Read [Aborting Random Oracles: How to Build them, How to Use them
(ePrint 2026/016)](https://eprint.iacr.org/2026/016).

---

## 6. Hash functions, tweaks, and security assumptions

### 6.1 Why tweaks are needed

XMSS invokes a hash function in many logically different contexts: chain steps, message hashing,
leaf compression, and every Merkle-tree level and position. Reusing the same raw hash input format
across those contexts can create structural ambiguity. A **tweak** domain-separates calls.

Typical tweak components include:

- epoch and chain index for a WOTS chain;
- position within a chain;
- Merkle level and node position;
- a separator identifying message hashing versus tree hashing.

The generalized framework assumes suitable multi-target collision resistance, preimage resistance,
and undetectability properties for the tweakable hash. The exact theorem is parameterized; the
engineering takeaway is that every hash call must use the intended domain and serialization.

### 6.2 Hashes inside aggregation proofs

The 2025/055 paper points out a proof-system tension. A verifier inside a succinct proof cannot
simultaneously treat a hash as an ideal random oracle and also expose its full circuit as an
ordinary algebraic computation. The construction therefore needs explicit standard-model hash
properties that can be reasoned about inside the proof. This is one reason the paper isolates the
encoding, tweakable hash, and multi-signature layers instead of treating “XMSS plus aggregation”
as one opaque primitive.

For this project, that distinction gives a useful boundary:

- signer-state tests check **when** a key is used;
- encoding tests check **which chain positions** a message maps to;
- hash/aggregation tests check **whether** the resulting proof or root is valid.

They are related, but one passing test cannot substitute for the others.

---

## 7. What `leanSig` implements today

The following is based on the `leanEthereum/leanSig` source and README, reviewed against commit
[`c08a3bae74b0d85379cab72dcbefa4091546ecbb`](https://github.com/leanEthereum/leanSig/tree/c08a3bae74b0d85379cab72dcbefa4091546ecbb). The repository is a moving prototype; verify the
current source before depending on a detail.

Start with the [README](https://github.com/leanEthereum/leanSig/blob/main/README.md) and the
[signature trait](https://github.com/leanEthereum/leanSig/blob/main/src/signature.rs).

### 7.1 Public API shape

The signature scheme exposes the conceptual operations:

```text
key_gen(rng, activation_epoch, num_active_epochs)
sign(secret_key, epoch, message)
verify(public_key, epoch, message, signature)
```

The secret key also exposes preparation operations because the implementation materializes only a
window of bottom Merkle trees:

```text
get_prepared_interval()
advance_preparation()
```

The trait requires serializable public keys, secret keys, and signatures. Canonical serialization
is SSZ-based in the current implementation. See
[`src/serialization.rs`](https://github.com/leanEthereum/leanSig/blob/main/src/serialization.rs).

### 7.2 The key/epoch uniqueness contract

The `SignatureScheme` documentation states that `sign` must not be called twice for the same
secret key, epoch, and **different message**. The implementation is deliberately deterministic:
the same key/epoch/message triple produces the same signature, because the PRF deterministically
derives the encoding randomness and chain starts.

That behavior has two important consequences:

1. An exact duplicate request can be safely made idempotent at the API layer.
2. Determinism does not make two different messages at the same epoch safe. The client must
   serialize, cache, reject, or otherwise guard the first signing decision.

`leanSig` reports encoding failure through
`SigningError::EncodingAttemptsExceeded { attempts }`. It does not mean that the key was
exhausted.

### 7.3 Activation and prepared intervals

`leanSig` separates two ranges:

- **Activation interval:** the complete range in which the key may be used.
- **Prepared interval:** the currently materialized window of bottom trees.

The prepared window is a sliding range of length `2C`, where
`C = sqrt(LIFETIME) = 2^(LOG_LIFETIME/2)`. Advancing preparation overlaps the old and new
windows by `C` leaves:

```text
old: [a, a + 2C)
new:       [a + C, a + 3C)
```

The overlap allows background preparation while the signer is still using the old window. A
request outside the activation interval is invalid. A request inside the activation interval but
outside the prepared window indicates that preparation has fallen behind. The current source uses
assertions around some of these preconditions, so a production adapter should convert them into a
controlled error rather than allowing a client process to panic.

### 7.4 Top-tree/bottom-tree memory optimization

Instead of keeping all `L` leaves and the whole Merkle tree in memory, the secret key stores:

- a top tree;
- the left bottom subtree; and
- the right bottom subtree.

When preparation advances, a new right bottom subtree is derived from the PRF key, the old right
subtree becomes the left subtree, and the window index moves forward. During the transition there
may be three bottom trees temporarily, but the old left subtree can then be dropped.

This optimization is important to test because it creates state transitions beyond “increment a
counter”: the client must not lose, repeat, or reorder the boundary between adjacent windows.

### 7.5 `leanSig` signing flow in source terms

The generalized XMSS implementation stores a secret PRF key, public parameters, activation
metadata, the top tree, the current bottom-tree index, and two bottom trees. Its `sign` method:

1. checks that the epoch is active and prepared;
2. selects the left or right bottom tree;
3. derives the combined Merkle path;
4. loops over encoding attempts;
5. derives `rho = PRF(prf_key, epoch, message, attempt_counter)`;
6. calls the incomparable encoding;
7. derives each chain start `PRF(prf_key, epoch, chain_index)`;
8. walks each chain for the selected digit; and
9. returns `path`, `rho`, and the chain values.

Verification recomputes the encoding, completes every chain, and verifies the Merkle path. See
[`generalized_xmss.rs`](https://github.com/leanEthereum/leanSig/blob/main/src/signature/generalized_xmss.rs).

### 7.6 Domain separation in the Poseidon instantiation

The Poseidon implementation uses distinct encodings for tree and chain contexts. The source
defines separate tweak forms for:

- Merkle tree level and position;
- epoch, chain index, and chain position; and
- message hashing.

The exact byte/field layout is part of the scheme definition. The source is the authority for
the current implementation:

- [Poseidon tweak hash](https://github.com/leanEthereum/leanSig/blob/main/src/symmetric/tweak_hash/poseidon.rs)
- [Tweakable hash trait](https://github.com/leanEthereum/leanSig/blob/main/src/symmetric/tweak_hash.rs)
- [Merkle tree implementation](https://github.com/leanEthereum/leanSig/blob/main/src/symmetric/tweak_hash_tree.rs)

### 7.7 Deliberate deviations from the paper

The README documents several implementation-level deviations from the paper’s notation:

- an overwrite sponge is used instead of an addition/XOR sponge for one public-key hash;
- the sponge layout is `[capacity | rate]` rather than `[rate | capacity]`;
- WOTS encoding input order is `[message, parameters, epoch, randomness]`;
- chain-hash input order is `[current value, parameter, tweak]`.

The project explains these as choices motivated by leanVM/XMSS aggregation performance and says
they do not change the intended security level. They are nevertheless interoperability details:
another implementation must match them exactly.

### 7.8 Current instantiation families

The Poseidon instantiation provides lifetimes `2^18` and `2^20`, target-sum encoding, and
Winternitz bases `w ∈ {1, 2, 4, 8}`. For `L = 2^18`, the source gives representative values:

| `w` | Dimension `v` | Target sum examples |
|---:|---:|---:|
| 1 | 155 | 78 / 86 |
| 2 | 78 | 117 / 129 |
| 4 | 39 | 293 / 322 |
| 8 | 20 | 2550 / 2805 |

The source warns that `w = 8` has long chains and high variance. The 2025/055 analysis generally
finds `w = 2` or `w = 4` to be the best size/work balance in its explored settings.

The aborting instantiations include a production-like `L = 2^32`, dimension `46`, base `8`,
target sum `200` configuration, and a deliberately small testing configuration with `L = 2^8`,
dimension `4`, base `8`, and target sum `6`. The small lifetime is particularly useful for
exhaustion tests, but it must never be mistaken for a production parameter set.

See the [Poseidon instantiations](https://github.com/leanEthereum/leanSig/blob/main/src/signature/generalized_xmss/instantiations_poseidon.rs)
and [aborting instantiations](https://github.com/leanEthereum/leanSig/blob/main/src/signature/generalized_xmss/instantiations_aborting.rs).

---

## 8. Parameters and the main tradeoffs

### 8.1 Lifetime and Merkle height

`L = 2^h` gives `L` signing positions and a Merkle authentication path of length `h`. Larger
lifetimes reduce key rotation frequency but increase tree construction and state-management
costs. `leanSig` splits the tree at half height to make bottom-tree preparation manageable.

The paper’s benchmarks explore `2^18` and `2^20`; a `2^32` lifetime is attractive for a long-lived
validator key but creates a much larger engineering and memory problem. The lifetime is also the
boundary that a test can intentionally shrink to make exhaustion reproducible.

### 8.2 Winternitz base and dimension

The chain digit range is `0 ... 2^w−1`.

- Smaller `w`: more coordinates and larger signatures, but shorter chains.
- Larger `w`: fewer coordinates and smaller signatures, but longer chain walks and potentially
  less predictable work.

This is not merely a bandwidth choice. It affects signing latency, verification latency, proof
circuits, memory, and how easy it is to exercise a test configuration.

### 8.3 Security levels

The 2025/055 paper uses a 128-bit classical / 64-bit quantum security target in its parameter
discussion. Quantum preimage search is treated with the usual square-root intuition, so the
quantum security target is not numerically identical to the classical one. Do not derive a new
parameter set by changing only the lifetime; the encoding dimension, hash output size, chain base,
retry count, and number of adversarial queries all enter the bound.

### 8.4 Signing retries versus verification work

The target-sum construction makes the verifier’s total chain work approximately fixed. Increasing
the target can reduce verifier work only at the cost of making valid encodings rarer, which means
more signer retries. In a consensus system, verification is performed by many nodes while signing
is performed by a proposer, so this asymmetry can be desirable—but only if retry behavior is bounded
and observable.

---

## 9. Failure modes that matter for this project

The following table separates cryptographic concepts that are easy to conflate.

| Failure mode | What happened | Correct client behavior | What a black-box test should observe |
|---|---|---|---|
| **Same epoch, different message** | One XMSS leaf signs two block roots | Refuse the second, or return the exact cached first result; never emit a second distinct valid signature | At most one distinct signature for `(key, epoch)` |
| **Exact duplicate request** | Same key, epoch, and message requested again | Idempotent cached result is acceptable | Same signature or an explicit duplicate response |
| **Encoding rejection** | `IncEnc(...)` returns `⊥` for one `rho` | Retry with bounded attempts | Retry is internal; do not advance the leaf twice |
| **Encoding attempts exhausted** | All allowed encodings reject | Return a structured signing error | No valid signature; process remains healthy |
| **Prepared-window miss** | Epoch is active but its bottom tree is not ready | Delay, prepare, or refuse in a controlled way | No panic; no use of the wrong adjacent leaf |
| **Activation violation** | Request is outside the key’s active interval | Reject | No signature for an invalid epoch |
| **Lifetime exhaustion** | No unused leaf remains | Refuse and rotate/replace the key | No signature beyond `L`; no counter wraparound |
| **Restart/persistence bug** | State is lost or rolled back after restart | Persist or recover consumed-position state | Restart cannot enable a second distinct signature |
| **Concurrency race** | Two requests pass a check before either marks the leaf used | Serialize or atomically commit one decision | Concurrent distinct messages still produce at most one signature |
| **Receiver-side equivocation handling** | A peer sends two valid sibling blocks | This is a fork-choice/consensus policy question, not the signer invariant | Keep separate from the signer-state test |

### Why exact replay is not enough

Because `leanSig` is deterministic for the same key/epoch/message, a test that sends the exact same
request twice may pass even if the implementation has no protection against conflicting messages.
The test must vary a commitment-bearing field such as the block root, payload root, or signing
message.

### Why an invalid-signature test is not enough

An XMSS signature can be individually valid even when the signer violated its state rule. The
receiver verifies the signature and Merkle path, not the signer’s private counter. Therefore, the
test needs access to the signer response, not only the receiving client’s block-import result.

---

## 10. Recommended Hive test boundary

### 10.1 Main test: same-epoch distinct-message signing

**Setup**

- Start a client with one XMSS validator key.
- Use one protocol epoch/slot.
- Construct two otherwise valid proposal messages with different block roots.

**Stimulus**

- Request a signature for message A.
- Request a signature for message B at the same epoch.
- Repeat with the requests issued concurrently if the driver can do so.

**Acceptable outcomes**

- A is signed and B is refused; or
- B is signed and A is refused; or
- the second request returns the exact cached result for the first message only if it is a true
  duplicate.

**Failure**

- Two different valid signatures verify under the same public key and epoch.

A useful assertion is:

```text
count(distinct valid signatures for (public_key, epoch)) <= 1
```

Do not require the receiver to reject a second block as the only assertion. That tests a different
boundary and may miss the signer bug.

### 10.2 Exhaustion test with a small-lifetime fixture

Use the small `L = 2^8` testing instantiation when available, or another deliberately tiny test
key. Sign every valid position once, then request one more signature.

Assert that:

- all positions in the configured interval can be signed;
- the first invalid position is refused or returns a documented exhaustion error;
- no valid signature is produced beyond the lifetime;
- the client remains alive; and
- a restart cannot reset the state and produce a second distinct signature for an old position.

This test catches off-by-one errors, counter wraparound, state rollback, and “exhaustion treated as
retry” bugs.

### 10.3 Preparation-boundary test

If the client exposes preparation, choose epochs around the boundary of the current `2C` window:

- last epoch in the left bottom tree;
- first epoch in the right bottom tree;
- first epoch after `advance_preparation()`;
- an active epoch not yet prepared.

The expected result is a correct signature for prepared epochs and a controlled wait/refusal for an
unprepared one. A signature that verifies under the wrong epoch or wrong Merkle path indicates a
leaf-selection or window-rotation bug.

### 10.4 Encoding-retry test (separate from state tests)

If the driver can force or observe the message-hash/encoding path, use a deterministic fixture to
exercise one or more rejected encodings. Verify that:

- retrying does not consume two epoch positions;
- the successful signature uses the requested epoch;
- `EncodingAttemptsExceeded` is surfaced distinctly from exhaustion; and
- a bounded retry limit prevents an infinite signing loop.

This is valuable, but it should not be presented as evidence that signer-state protection works.

---

## 11. What the signer operation needs to expose

The peer-injected block path is not sufficient for the main test: injecting two blocks into a
client exercises block reception, not the client’s own XMSS signer. The test harness needs a
small signer operation or client adapter that can request and observe signing.

The exact shared schema should be agreed with the client maintainers, but a useful minimum is:

### Request

```text
{
  validator_id,
  public_key_or_key_id,
  epoch_or_slot,
  signing_root_or_block_root,
  request_id
}
```

### Response

```text
{
  request_id,
  status: signed | refused | error,
  epoch_or_slot,
  signing_root_or_block_root,
  signature,
  error_code,
  error_message
}
```

The driver should not require clients to reveal private counters. It only needs enough information
to verify returned signatures and compare `(validator, epoch, message)` pairs. Optional diagnostic
fields—selected leaf index, retry count, prepared interval, or exhaustion state—are useful for
debugging but should not be required for interoperability.

The contract should define:

- whether the operation is synchronous or returns a job ID;
- what an exact duplicate request returns;
- how a conflicting same-epoch request is reported;
- whether concurrent requests are allowed;
- how activation, preparation, encoding, and exhaustion errors are distinguished; and
- whether state must survive a client restart.

Keeping these outcomes explicit prevents a Hive test from interpreting every non-signature response
as “XMSS rejected the request” when the real cause was an encoding retry or a preparation delay.

---

## 12. What is already covered versus what is novel

There are three layers of coverage:

1. **Cryptographic verification:** does a signature open to the public Merkle root and verify for
   the stated message and epoch?
2. **Consensus reception:** how does a client store, validate, and choose among valid blocks?
3. **Signer state:** does the client prevent two different messages from consuming the same
   one-time signing position?

Existing XMSS/LeanSpec-style signature verification tests can cover layer 1. Receiver-side tests
can cover layer 2. Neither automatically covers layer 3, because a receiver sees only the valid
signatures supplied to it. The value of the proposed Hive work is to exercise the real signer
operation through a common black-box interface.

This distinction also clarifies the risk: if layer 3 is not tested, a client can pass all ordinary
signature-validation tests while a counter, cache, persistence, or concurrency bug still allows a
real validator to create two valid proposals at one slot.

---

## 13. Paper-by-paper takeaways

### 13.1 2025/055 — Hash-Based Multi-Signatures for Post-Quantum Ethereum

Use this paper for:

- the Ethereum motivation for replacing BLS with hash-based signatures;
- the generalized XMSS construction;
- strong unforgeability rather than only ordinary existential unforgeability;
- randomized incomparable encodings and their target-collision notion;
- target-sum Winternitz encoding;
- tweakable-hash assumptions;
- parameter/security guidance; and
- the aggregation/proof-system context.

The project-specific lesson is that aggregation does not remove the statefulness requirement. A
proof that many individual signatures are valid still needs each individual signer to obey its
one-epoch/one-leaf rule.

Source: [ePrint 2025/055](https://eprint.iacr.org/2025/055),
[IACR publication](https://cic.iacr.org/p/2/1/13).

### 13.2 2025/889 — At the Top of the Hypercube

Use this paper for:

- why the encoding geometry controls verification work;
- a lower bound for general encodings at fixed signature size;
- non-uniform encodings into top layers of a larger hypercube; and
- the reported 20–40% verification-cost improvement.

It is relevant when discussing future signature-size or verifier-performance work. It is not a
replacement for signer-state testing and should not be used to describe duplicate signing.

Source: [ePrint 2025/889](https://eprint.iacr.org/2025/889).

### 13.3 2026/016 — Aborting Random Oracles

Use this paper for:

- reasoning about message hashes that reject a small fraction of inputs;
- the aborting random-oracle model;
- translating aROM reasoning back to standard-ROM guarantees; and
- proof-friendly incomparable hypercube encodings.

It is relevant to retry behavior and proof-oriented implementations. An abort is not a used leaf,
and an abort is not exhaustion.

Source: [ePrint 2026/016](https://eprint.iacr.org/2026/016).

### 13.4 `leanSig`

Use the repository for:

- the actual Rust trait and error surface;
- deterministic PRF-derived signing randomness;
- activation/preparation intervals;
- top-tree/bottom-tree preparation;
- Poseidon domain separation and serialization;
- target-sum and aborting instantiations; and
- executable tests and parameter aliases.

Sources: [repository README](https://github.com/leanEthereum/leanSig/blob/main/README.md),
[signature API](https://github.com/leanEthereum/leanSig/blob/main/src/signature.rs),
[generalized XMSS implementation](https://github.com/leanEthereum/leanSig/blob/main/src/signature/generalized_xmss.rs).

For classical XMSS terminology and the standard wire-level construction, consult
[RFC 8391](https://www.rfc-editor.org/rfc/rfc8391). The Lean/Ethereum constructions are not
necessarily byte-for-byte compatible with RFC 8391, so use the RFC as background, not as an
implementation substitute.

---

## 14. Practical checklist for reviewing an XMSS client

### Key lifecycle

- Is the activation interval explicit and validated?
- Is the lifetime bounded and checked before incrementing a position?
- Is the current signing position persisted before or atomically with returning a signature?
- What happens after restart, crash recovery, or rollback?

### Same-epoch protection

- Is the key/epoch/message decision cached?
- Does a conflicting message get refused before signing?
- Is the check-and-mark operation atomic under concurrent requests?
- Can two worker threads select the same leaf before either updates state?

### Preparation

- Is the prepared interval visible to the signer?
- Are left/right bottom-tree boundaries tested?
- Does advancing the window preserve the overlap correctly?
- Can a stale worker sign using a bottom tree that has already moved out of the active window?

### Encoding and hashing

- Are encoding rejects retried with bounded attempts?
- Is `rho` derived consistently for `(key, epoch, message, attempt)`?
- Are message, chain, and Merkle-tree tweaks domain-separated?
- Are parameter, epoch, and randomness serialized in exactly the specified order?
- Does verification complete the correct number of chain steps?

### API and observability

- Can a black-box test request a proposal signature directly?
- Can it distinguish signed, duplicate, refused, encoding, preparation, and exhausted outcomes?
- Can it verify the returned signature independently?
- Does the client remain healthy after every expected refusal?

---

## 15. Suggested terminology for the project proposal

Use precise language such as:

> **XMSS signer-state non-reuse:** exercise repeated and concurrent proposal attempts for one
> validator key and one protocol slot, and verify that the client produces at most one distinct
> valid proposal signature for that slot.

> **Key exhaustion:** exercise the first signing position beyond a deliberately small test lifetime
> and verify a controlled refusal without counter wraparound or process failure.

> **Preparation-boundary safety:** exercise epochs at adjacent bottom-tree windows and verify that
> preparation advances without selecting the wrong leaf or Merkle path.

Avoid saying that the test “checks that the second block is rejected” unless the test is explicitly
about receiver-side block processing. For signer-state testing, the correct assertion is that the
client does not create two distinct valid signatures.

---

## 16. One-page mental model

```text
                 long-lived XMSS public key
                         Merkle root
                              |
       -------------------------------------------------
       |        |        |        |                  |
    leaf 0   leaf 1   leaf 2   leaf 3              leaf L-1
       |        |        |        |
    epoch 0  epoch 1  epoch 2  epoch 3       (one leaf per signing position)
       |
   WOTS-like chains + authentication path + rho
       |
   one valid proposal signature for this key/epoch
       |
   state must prevent a second distinct message at the same epoch
```

When debugging a failure, ask in this order:

1. Did the message encoding reject, or did the key have no valid position?
2. Which protocol epoch/slot was requested?
3. Which XMSS leaf did the client map it to?
4. Was a signature actually returned, and does it verify?
5. Was another distinct valid signature returned for the same key and epoch?
6. Did preparation, persistence, or concurrency make the answer different after restart or
   parallel requests?

That sequence keeps the project focused on its central risk: **stateful signing failures in a
post-quantum Ethereum client**, not merely whether an isolated XMSS signature can be verified.
