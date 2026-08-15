# EPF Cohort 7 Project Handoff

## XMSS signer-state testing in Ethereum Hive

**Fellow:** Mohit Grover  
**Current direction:** black-box testing of XMSS signer-state failure modes in the Lean Consensus
simulator in [`ethereum/hive`](https://github.com/ethereum/hive)  
**Primary initial client:** Ream  
**Status date:** 15 August 2026

This document contains the project context needed to continue the work in a new chat without
repeating earlier research, incorrect assumptions, or already completed work.

---

## 1. Executive summary

The project is now focused on one concrete safety problem introduced by Ethereum's proposed move
from reusable BLS keys to stateful XMSS signing keys:

> A validator client must never produce two distinct valid proposal signatures using the same XMSS
> key and signing epoch/slot.

The core contribution is not another receiver-side block-validation test. It is a Hive mechanism
that exercises a client's **own signer**, observes the resulting signature, and tests whether the
client manages the stateful XMSS lifecycle safely.

The two central failure classes are:

1. **Signing-material reuse:** two different block roots are signed for the same key and slot.
2. **Key exhaustion:** the client attempts to sign after the key has no valid signing positions
   remaining.

The first prerequisite is a common test-only signer operation or client adapter. The existing
Hive `libp2p_mock` can inject and capture gossip, but peer-injected blocks do not exercise the
client's signing boundary.

The earlier validation-suite work has been completed and merged. It provided useful Lean simulator
infrastructure, fixed Ream devnet5 support, and exposed several harness-level pitfalls, but it is
foundation work rather than the final XMSS deliverable.

---

## 2. Project direction

### Current project statement

Build black-box Hive tests that make stateful XMSS signer failures observable across Lean Consensus
clients, beginning with Ream.

### Why this matters

BLS keys are reusable. XMSS is built from a finite set of one-time signing positions committed
under a Merkle root. A valid XMSS signature does not prove that the signer has not already used the
same position for another message. Therefore, ordinary signature-verification and block-import tests
can all pass while a validator still has an unsafe counter, cache, persistence, preparation, or
concurrency implementation.

This creates a new validator-safety surface that existing Ethereum tests were not designed to
exercise.

### Core invariant

For a public key `pk` and signing epoch/slot `e`:

```text
count(distinct valid signing messages returned by the client for (pk, e)) <= 1
```

The exact same request may return the same deterministic signature again. The prohibited outcome is
a second valid signature for a **different** message or block root at the same signing position.

### Intended testing layer

```text
Hive test driver
      |
      | signer request: key + slot/epoch + signing root
      v
client signer / validator layer
      |
      | signed | refused | structured error
      v
Hive independently verifies and compares the response
```

The test should observe externally visible behavior. It should not depend on a client's internal
counter representation.

---

## 3. Scope

### Core scope

- Define a common signer request/response contract or a small per-client adapter.
- Exercise two different proposal messages at the same signing slot.
- Test sequential and, if possible, concurrent requests.
- Test the first request beyond a deliberately small XMSS lifetime.
- Verify that refusals and exhaustion do not crash the client.
- Make the test reusable by additional Lean clients when they expose the required operations.

### Strong follow-up coverage

- Restart after signing and retry the same slot.
- Restore stale signer state and confirm that reuse is still prevented.
- Exercise XMSS prepared-window/bottom-tree boundaries.
- Cross-client signature verification when at least two clients share a compatible wire format.

### De-prioritized or outside the present XMSS-only direction

- Requiring a receiving client to reject the second of two valid equivocating blocks.
- A complete XMSS cryptographic conformance suite inside Hive.
- Intermediate PRF, WOTS-chain, or Merkle-node test vectors unless a stable common format exists.
- Production secret-key export/import APIs.
- Signature aggregation proofs before concrete client interfaces exist.
- Checkpoint-adversarial fixtures, live-head state comparison, and broad state-transition auditing.
- Multiple loosely related stretch goals merely to fill the 16-week roadmap.

An older proposal revision included a finalized-state cross-client comparison as a single stretch
goal. Under the latest “entirely XMSS” direction, it should remain optional and must not displace
the signer-state work.

---

## 4. Milestones already achieved

### 4.1 Problem-space research and scope reduction

Three areas were initially considered:

- XMSS lifecycle and signer state;
- signature aggregation; and
- state-transition/checkpoint comparison.

Mentor feedback led to narrowing the core scope to XMSS lifecycle failures. This was the correct
choice because it has a precise safety invariant, an identifiable gap in Hive, and a deliverable
that does not require the full aggregation stack to exist.

### 4.2 Proposal correction: equivocation belongs at the signer boundary

The initial idea described publishing two differently signed blocks at the same slot and expecting
the receiving client to reject the second one. Investigation of LeanSpec showed that this expectation
was wrong.

LeanSpec's equivocation test permits two different, correctly signed blocks from the same proposer at
the same slot to be valid and stored. Fork choice chooses one deterministic head. There is no general
receiver-side rule saying “a block from this proposer and slot has already been seen, therefore reject
the next root.”

Consequences:

- The two blocks are competing unfinalized branches; neither should be called “finalized.”
- Receiver acceptance is not proof of an XMSS implementation bug.
- Hive supplying both signatures only proves that an equivocation was presented.
- To test XMSS statefulness, Hive must cause the client's signer to attempt both proposals and
  observe whether it emits two distinct signatures.

This was the project's most important technical correction.

### 4.3 Hive validation PR merged

[Hive PR #1590](https://github.com/ethereum/hive/pull/1590) was merged as commit
`d764a937a23defce29c2e772843eb55e023c4a6b`.

It added three validation scenarios:

1. Reject a block beyond the future-slot horizon.
2. Accept and produce a valid block after receiving invalid gossip.
3. Treat an exact duplicate valid block as idempotent.

The suite already contained invalid-proposer, invalid-parent-root, and invalid-state-root checks.
The resulting validation suite now contains all six scenarios plus the client-launch test.

The PR also changed supporting files:

- [`simulators/lean/src/scenarios/validation.rs`](../simulators/lean/src/scenarios/validation.rs)
- [`simulators/lean/src/utils/libp2p_mock.rs`](../simulators/lean/src/utils/libp2p_mock.rs)
- [`clients/ream/Dockerfile`](../clients/ream/Dockerfile)
- [`clients/ream/ream.sh`](../clients/ream/ream.sh)

### 4.4 Ream devnet5 image fixed and devnet4 removed

The validation work initially could not run the current Ream devnet5 binary because its glibc
requirements were newer than the old runtime image. The final merged Docker setup:

- removes obsolete devnet4 support;
- uses the Ream devnet5 image/tag;
- copies the Ream binary into an Ubuntu 24.04 runtime;
- installs the required CA certificates; and
- makes devnet5 the default in `ream.sh`.

This was an image/runtime compatibility issue, not evidence that Hive maintainers had intentionally
kept devnet4 as the current supported network.

### 4.5 Gossip harness follow-up merged

The initial Ream devnet5 run passed every validation scenario except
`duplicate valid block is idempotent`.

The failure was not a Ream consensus bug. The mock publisher's own gossipsub duplicate cache rejected
an attempt to publish the exact same bytes before the message reached Ream. This taught an important
lesson: a failing Hive assertion can originate in the harness, transport, framing, or client, and
those layers must be isolated before assigning blame.

[Hive PR #1594](https://github.com/ethereum/hive/pull/1594), merged as commit `f6e021bb`, fixed:

- mock-side duplicate publishing by changing the mock-local message-ID salt while retaining the
  payload bytes;
- gossip Snappy framing; and
- the related message capture/replay behavior.

The current upstream `libp2p_mock` can capture a client-produced signed envelope and replay its exact
payload, including its XMSS/aggregate proof.

### 4.6 XMSS background research completed

A detailed local reference was created:

- [XMSS background knowledge](xmss-background-knowledge.md)

It covers XMSS/WOTS chains, Merkle authentication paths, incomparable and target-sum encodings,
aborting encodings, `leanSig` internals, prepared intervals, failure modes, and recommended Hive
assertions.

The key research sources are:

- [Hash-Based Multi-Signatures for Post-Quantum Ethereum](https://eprint.iacr.org/2025/055)
- [At the Top of the Hypercube](https://eprint.iacr.org/2025/889)
- [Aborting Random Oracles](https://eprint.iacr.org/2026/016)
- [`leanEthereum/leanSig`](https://github.com/leanEthereum/leanSig)

---

## 5. Main technical findings

### 5.1 `leanSig` does not itself provide durable reuse protection

The `leanSig` signing interface documents that the same secret-key/epoch pair must not be used for
different messages. Its generalized signer deterministically derives:

- chain starts from the PRF key, epoch, and chain index; and
- encoding randomness from the PRF key, epoch, message, and retry counter.

Therefore:

- same key + same epoch + same message produces the same signature;
- same key + same epoch + different message can produce a second, different valid signature; and
- the caller/client integration is responsible for durable state protection.

The project may therefore expose a missing production wrapper or client-level safeguard rather than
a bug in the cryptographic `leanSig` library. That is still a valid and important outcome.

### 5.2 Slot, epoch, and leaf index must not be conflated

The protocol request contains a slot or epoch. XMSS uses a leaf/signing-position index. An integration
may apply an activation offset or another mapping.

Do not assume:

```text
protocol slot 42 == XMSS leaf 42
```

The common signer contract must define the mapping or expose enough metadata to diagnose it.

### 5.3 Exact replay is not the dangerous case

Because signing is deterministic for the same triple, an exact duplicate request can safely return
the exact cached signature. A useful test must change the signing root or another commitment-bearing
part of the proposal.

Correct assertion:

```text
at most one distinct valid signature for (key, slot/epoch)
```

Incorrect assertion:

```text
the second API call must always fail
```

### 5.4 Signature validity does not reveal leaf reuse

Each of two conflicting signatures may verify individually. The verifier reconstructs the WOTS
endpoint and Merkle root; it does not read the signer's private consumed-leaf database.

Consequently, cryptographic verification alone cannot test signer state.

### 5.5 Encoding retry is not key exhaustion

Target-sum or aborting message encodings may reject a candidate `rho`, after which signing retries.
`leanSig` can report `EncodingAttemptsExceeded` when no encoding succeeds within the bound.

Keep these outcomes separate:

- encoding retry/failure;
- duplicate signing position;
- prepared-window miss;
- activation-range violation; and
- lifetime exhaustion.

Tests and the signer response schema should not collapse all of them into a generic “XMSS error.”

### 5.6 Prepared state is more than a counter

`leanSig` uses a top-tree/bottom-tree design. The prepared interval is a sliding two-bottom-tree
window. Moving the window changes cached Merkle material and creates boundary risks:

- wrong adjacent bottom tree;
- off-by-one at the window edge;
- stale worker using a retired subtree;
- crash during preparation advancement; and
- active epoch requested before its tree is ready.

Prepared-window testing is worthwhile after the basic signer operation works.

### 5.7 Safety requires persistence and atomicity

The strongest operational risks are:

- two concurrent requests passing a non-atomic “unused” check;
- restart losing the consumed-epoch record;
- stale snapshot restoration;
- two processes/devices using the same key; and
- migration that copies the key but not its state.

A safety-first implementation should durably reserve a signing position before returning a
signature. A richer design stores `(key, epoch, signing_root, state, signature)` so an exact request
can be recovered while a conflicting request remains prohibited.

A crash after reservation may sacrifice one signing opportunity. That is a liveness loss, but it is
safer than reusing the position.

---

## 6. Common signer operation: current design direction

### Why it is required

`libp2p_mock` exercises networking and block reception. Publishing a signed block to a client does
not ask that client to use its own XMSS key. The core tests require a new test-driver hook or
per-client adapter that calls the validator/signer path and returns the signed result.

This is not merely another method inside `libp2p_mock`. The mock may transport or observe the
result, but the operation must cross into the client's signer.

### Suggested minimum request

```json
{
  "request_id": "...",
  "validator_id": "...",
  "key_id": "...",
  "slot_or_epoch": 42,
  "signing_root": "0x..."
}
```

### Suggested minimum response

```json
{
  "request_id": "...",
  "status": "signed | refused | error",
  "slot_or_epoch": 42,
  "signing_root": "0x...",
  "signature": "0x...",
  "error_code": "...",
  "error_message": "..."
}
```

### Required semantic decisions

- Is an exact duplicate returned from a cache, recomputed deterministically, or refused?
- What is returned for a different message at an already consumed slot?
- Are concurrent requests supported?
- Is the signing decision persisted before the response is returned?
- How are activation, preparation, encoding, exhaustion, and internal errors distinguished?
- Does the behavior survive client restart?
- Is `slot_or_epoch` directly the XMSS leaf index or translated by the client?

### Optional diagnostics

These should help debugging but should not be interoperability requirements:

- selected leaf index;
- encoding retry count;
- prepared interval;
- key activation interval; and
- exhaustion state.

### Collaboration with Richard

The proposal should remain explicit that shared test-driver response conventions are being
coordinated with Richard. The useful collaboration boundary is the envelope and error semantics,
not forcing both projects to implement identical internal logic.

Before implementation, record:

- which fields both projects need;
- who owns the shared Rust types or schema;
- which client adapters will implement it first; and
- which milestone depends on the other person's work.

Do not leave this as an informal “we will collaborate” statement; make the dependency and ownership
visible in the roadmap or issue tracker.

---

## 7. Recommended tests and exact success criteria

### 7.1 Sequential same-slot distinct-message test

1. Start a client with one test validator key.
2. Request a proposal signature for slot `e` and signing root `A`.
3. Request another signature for the same key and slot `e`, but root `B`.
4. Independently verify every returned signature.

Pass when no more than one distinct valid `(root, signature)` pair is returned.

Acceptable client behavior:

- sign `A`, refuse `B`;
- sign `B`, refuse `A`; or
- for an exact duplicate only, return the cached original result.

Fail when valid signatures for both roots are returned.

### 7.2 Concurrent same-slot distinct-message test

Issue the two requests concurrently. This is necessary because a sequential test can pass while a
check-then-write race still exists.

Use the same success criterion: at most one distinct valid signature.

### 7.3 Restart persistence test

1. Sign root `A` at slot `e`.
2. Stop and restart the client without intentionally deleting its state.
3. Request root `B` at slot `e`.

Pass when the client does not return a valid signature for `B`.

### 7.4 Exhaustion test

Use a deliberately small, test-only XMSS lifetime if the client can configure one. `leanSig` has a
small testing instantiation, but that does not mean every client currently exposes it.

1. Sign every valid position once.
2. Request the first position outside the lifetime or activation interval.
3. Verify a controlled refusal and continued client health.

Check specifically for:

- off-by-one at the last valid position;
- counter wraparound;
- reuse of the first leaf;
- panic/process exit; and
- confusing encoding failure with exhaustion.

### 7.5 Preparation-boundary test

Once the signer adapter supports the relevant key type, exercise positions around the bottom-tree
boundary `C`:

```text
C - 1, C, C + 1, 2C - 1, 2C, 2C + 1
```

Verify correct signatures for prepared positions and controlled behavior for an unprepared active
position.

### 7.6 Cross-client sign/verify

Only add this when at least two clients implement compatible parameters, serialization, and a
verification interface.

Run both directions:

```text
client A signs -> client B verifies
client B signs -> client A verifies
```

Public-key and signature interoperability are sufficient. Secret-key import/export should not be
required unless a standard format is explicitly intended.

---

## 8. Immediate next steps

### Step 1: produce a client capability matrix

Audit Ream first, then every other active Lean client. For each client record:

| Capability | Questions to answer |
|---|---|
| Signer entry point | Can a test request the client's own proposal signature? |
| Key setup | Can Hive load a deterministic test validator key? |
| Signature output | Can Hive retrieve the exact signature and signing root? |
| Verification | Is there a verifier API, or should Hive use `leanSig` directly? |
| Parameters | Which XMSS lifetime/base/dimension are supported? |
| Slot mapping | How is protocol slot/epoch mapped to the XMSS position? |
| Persistence | Where is consumed signer state stored? |
| Restart | Does a normal restart retain the state? |
| Concurrency | Can more than one signing request execute concurrently? |
| Preparation | Who advances the prepared interval, and when? |
| Errors | Are duplicate, exhausted, unprepared, and encoding failures distinct? |

Mark every cell with evidence: source path, API route, command, or maintainer confirmation. Do not
infer support because a client imports `leanSig`.

### Step 2: define the signer-operation contract

Write the minimal shared schema and semantics before changing Hive. Resolve the questions listed in
Section 6 and review them with mentors, Richard, and the first client maintainer.

### Step 3: build the smallest Ream adapter/prototype

Exercise one signing request end-to-end:

```text
Hive -> Ream signer -> signed response -> independent verification
```

Do not begin with concurrency or exhaustion until one ordinary request is observable and verifiable.

### Step 4: implement same-slot non-reuse

Add the sequential different-root case, then the concurrent variant. These are the highest-value
tests and the clearest proof that the project is testing signer state rather than receiver behavior.

### Step 5: implement exhaustion

Determine whether Ream can use a small test lifetime without changing production parameters. If not,
agree on a safe test-only build flag, fixture, or injected signer backend.

### Step 6: add restart/persistence coverage

Once Ream's state location and restart behavior are understood, add the restart case. Stale snapshot
restoration can follow if Hive has a controlled way to copy or replace only the test state.

### Step 7: generalize to a second client

Only after the Ream path is stable should the interface be treated as cross-client. A second
implementation is what distinguishes an adapter-specific test from a true Hive interoperability
contract.

---

## 9. Questions that still need answers

1. Which currently active Lean clients actually use `leanSig` for proposal signing?
2. Does Ream expose a signer operation, or must a test-only adapter be added?
3. Where does Ream enforce same-key/same-epoch non-reuse, if anywhere?
4. Is signer state durable across an ordinary container restart?
5. What exact value is treated as the XMSS epoch: protocol slot, relative activation index, or
   another derived value?
6. Can a small-lifetime test key be configured without maintaining a divergent cryptographic fork?
7. Which component should independently verify the returned signature in Hive?
8. What response should clients return for exact duplicates versus conflicting messages?
9. How should the shared schema with Richard be owned and versioned?
10. Which client will be the second implementation after Ream?

These questions should be answered from code or maintainer confirmation before presenting a test as
cross-client compatible.

---

## 10. Mistakes and dead ends not to repeat

### Do not test receiver rejection as XMSS state protection

Two valid same-slot blocks can both be accepted and stored by LeanSpec. The signer must be tested
directly.

### Do not call competing blocks finalized

They are competing unfinalized branches. Fork choice choosing one head is not finalization.

### Do not use the exact same message for the non-reuse test

Deterministic signing may return the same signature and hide the dangerous case. Change the signing
root.

### Do not treat a supplied signature as evidence that the client created it

Hive or a mock peer injecting a signed envelope only tests reception unless the signature came from
the client's signer operation.

### Do not conflate encoding aborts with key exhaustion

Encoding can retry at one epoch. Exhaustion means no valid signing position remains.

### Do not assume slot equals leaf index

Confirm the client's activation and mapping rules.

### Do not blame the client before isolating the harness

The duplicate-gossip failure was caused by the mock publisher's duplicate cache. Check:

1. test construction;
2. serialization/compression;
3. gossip publication and delivery;
4. client logs and observable state; and
5. assertion logic.

### Do not require secret-key interoperability without a standard

Cross-client public-key/signature verification is enough for the first interoperability suite.

### Do not design an enormous common API first

Start with one request, one signed/refused/error response, and independent verification. Add
diagnostics only when a real test needs them.

### Do not overfill the roadmap with unrelated stretch goals

The core signer hook and two stateful failure tests are substantial. Broader checkpoint,
state-transition, and aggregation work should not make the central deliverable look unrealistic.

---

## 11. Risks to track

| Risk | Impact | Mitigation |
|---|---|---|
| No client exposes its own signer | Core test cannot run | Scope a test-only adapter early |
| Only Ream implements the operation | Test is client-specific | Stabilize semantics, then recruit a second client |
| State protection belongs outside `leanSig` | Library tests pass while client is unsafe | Test the production caller/wrapper boundary |
| Small lifetime is unavailable | Exhaustion is impractical to reach | Add a clearly test-only parameter/fixture |
| Determinism masks reuse | Duplicate test gives false confidence | Use two different signing roots |
| Concurrency is serialized by the test adapter | Race is not exercised | Confirm requests overlap at the real signer boundary |
| Restart recreates rather than preserves the client state | Persistence test is meaningless | Explicitly control which state volumes survive |
| Parameter/wire formats differ across clients | Cross-verification fails for expected reasons | Establish a compatibility matrix before asserting interop |
| Prepared-window assertion panics | Availability failure | Require adapters to turn precondition failures into structured responses |
| Harness transport failure looks like client rejection | False diagnosis | Assert publication/delivery and inspect client-observable state |

---

## 12. Repository map and current state

### Important files

- [`simulators/lean/src/scenarios/validation.rs`](../simulators/lean/src/scenarios/validation.rs) — merged validation scenarios.
- [`simulators/lean/src/utils/libp2p_mock.rs`](../simulators/lean/src/utils/libp2p_mock.rs) — Lean gossip/request-response mock, signed-envelope capture, duplicate replay support, SSZ types.
- [`clients/ream/Dockerfile`](../clients/ream/Dockerfile) — devnet5-only Ream image and compatible runtime.
- [`clients/ream/ream.sh`](../clients/ream/ream.sh) — devnet5 launcher.
- [`docs/xmss-background-knowledge.md`](xmss-background-knowledge.md) — detailed XMSS research reference.

### Relevant merged commits

- `d764a937` — Hive PR #1590, validation tests and infrastructure, Ream devnet5 cleanup.
- `f6e021bb` — Hive PR #1594, duplicate-publish and gossip Snappy framing fixes.

### Worktree status at handoff

The repository is on `master` at upstream commit `654c734f`. The XMSS knowledge document is currently
untracked:

```text
?? docs/xmss-background-knowledge.md
```

This handoff document is also newly created by the present task. Do not accidentally delete or
overwrite either file during future branch changes.

---

## 13. Weekly-update references

Previous EPF updates, useful for matching Mohit's writing style and reconstructing chronology:

- <https://hackmd.io/SlyloPmCTiOGIbm1mNhKbA>
- <https://hackmd.io/0iNmC60cT-CCxuDuJhd8_A>
- <https://hackmd.io/Sv3npwsdRZG_nS11hgtddw>

When writing future updates, emphasize reasoning:

- what uncertainty was investigated;
- what evidence changed the project direction;
- why a test belongs at a particular boundary;
- what a failure would mean; and
- what dependency is being retired next.

Avoid presenting a list of changed files as the main progress narrative.

---

## 14. Recommended opening context for the next chat

The following short prompt can be pasted at the beginning of a new conversation:

> I am working on an EPF Cohort 7 project adding black-box XMSS signer-state tests to the Lean
> simulator in ethereum/hive, starting with Ream. The validation foundation was merged in Hive PRs
> #1590 and #1594. The core invariant is that one validator key and one slot/epoch must produce at
> most one distinct valid proposal signature. Receiver-side acceptance of two equivocating blocks
> is not the test; Hive must exercise the client's own signer. The next task is to audit current Lean
> clients for signer APIs, define a small common signer request/response contract with Richard, and
> prototype one end-to-end Ream signing request before implementing sequential/concurrent non-reuse,
> exhaustion, and restart tests. Read `docs/epf-project-handoff.md` and
> `docs/xmss-background-knowledge.md` before proposing changes.

---

## 15. Definition of project success

The project is successful when Hive can run against a real Lean client and demonstrate, through a
documented external interface, that:

1. Hive can request and independently verify a client-produced XMSS proposal signature.
2. Two different proposals for the same key and slot never yield two distinct valid signatures,
   including under concurrent requests if supported.
3. The client handles an exhausted test key without wraparound, unintended reuse, or process failure.
4. The interface and test semantics are documented well enough for another Lean client to implement.
5. Results clearly distinguish client bugs, missing signer wrappers, unsupported capabilities, and
   Hive harness failures.

That is a focused, technically meaningful EPF deliverable. Everything else should support these
five outcomes rather than compete with them.
