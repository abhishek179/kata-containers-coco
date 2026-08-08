# FR-1: retiring the bespoke fragment signing statement (design note)

**Status:** design note. Not implemented, not scheduled. Written so the decision is on
record with its costs, because the cheap half has already landed and the expensive half
keeps looking cheaper than it is.

**Prerequisite:** RM-73 (F-151), which closed the envelope-free delivery path. This note is
only actionable now that the guest requires a COSE_Sign1 envelope.

---

## 1. The divergence

C-ACI/hcsshim and this branch both deliver a signed policy fragment as a COSE_Sign1
envelope over an OCI artifact. They disagree about what is *inside* it.

| | C-ACI / hcsshim | kata (today) |
| --- | --- | --- |
| COSE payload | the Rego module | a bespoke **statement** that wraps the module |
| issuer / feed / SVN | COSE protected headers + CWT claims | inside the payload |
| what the signature covers | headers + module, via the COSE `Sig_structure` | the statement bytes |

The kata form requires a hand-rolled, canonical, injective encoding of eight fields, plus
a decoder for it in the guest. C-ACI needs neither: COSE already defines how headers are
encoded and what the signature covers.

## 2. Why the statement exists at all

Not by preference. `LoadPolicyFragment` used to accept a fragment described by individual
request fields, with a **detached** signature and no envelope. A detached signature needs
something to sign, and "the COSE protected headers" is not available when there is no COSE.
So a signable byte string had to be invented.

Every defect found in that byte string traces to that one requirement:

| Finding | Defect | Fixed by |
| --- | --- | --- |
| F-144 | delimiter injection across concatenated fields | RM-68 (constrain the domain) |
| F-145, F-146 | ambiguity between field values and separators | RM-69 (escape `id` components) |
| F-144 residual | the format was still delimiter-based | RM-71 (re-encode as CBOR v4) |

Three rounds of hardening on a format that exists to serve a path nothing in production
used. RM-73 removed that path. **The statement is now load-bearing for nothing but its own
history.**

## 3. What retiring it would look like

Move the signed metadata into the COSE protected header and make the payload the Rego
module, matching hcsshim:

- `PolicyFragment::signing_bytes` deleted; the signature target becomes the COSE
  `Sig_structure`, which `coset` already builds.
- `from_cose_payload` replaced by a protected-header reader (issuer, feed, SVN, grants,
  includes, requires, `prev_log_head`), with the payload taken as the module.
- `verify_cose` / `verify_cose_x509` stop comparing the payload to a statement.
- Header labels chosen to match hcsshim's, so `sign1util` and `az confcom` produce
  fragments this guest accepts without a payload-construction step.

### What it buys

1. **Real tooling interop.** Today an external signer can sign our payload — `sign1util`
   treats it as an opaque file — but something must first build the CBOR statement, and
   only the SRM crate knows how. After this, an existing C-ACI signing pipeline produces a
   fragment this guest accepts.
2. **F-150 closes for free.** The OCI layer is published as
   `application/cose-x509+rego` (`genpolicy-fragmentgen/src/main.rs:39`) under artifactType
   `application/x-ms-ccepolicy-frag` — both copied from C-ACI — while the payload is *not*
   rego. A media type is a parsing contract, so an hcsshim consumer would accept the tag
   and misparse. Making the payload actually be rego makes the tag true.
3. **A whole defect class disappears.** No bespoke format means no injectivity gate, no
   canonicality check, no escaping rules, and no fourth round of the above table.
4. **One less thing to specify.** "Deterministically encoded CBOR in COSE protected
   headers" is a standard. Our statement is a spec we would have to write, version and
   defend to any third-party reviewer.

## 4. Why it has not been done

**FR-1f receipts and FR-1j ordering both countersign the statement bytes.** They are not
incidental readers of the format; they are built on it.

- A transparency receipt is an independent signature **over the same statement** by a
  ledger key (`fragments.rs:2001`), and `extra_receipts` are further countersignatures over
  that identical byte string. A CCF receipt binds the statement explicitly — the negative
  test signs `b"different-statement"` to prove the binding is checked
  (`fragments.rs:2824`).
- The ordering log chains `sha256(prev_head || sha256(statement))`, and a fragment's
  `prev_log_head` must equal the store's current head (`fragments.rs:1374`), which is what
  rejects reordering, omission and insertion.

Both need a stable, agreed byte string identifying a fragment. Retiring the statement means
re-basing them onto something else — the natural candidate being the COSE_Sign1 bytes, or
the `Sig_structure`. That is a defensible change, but it is a **design change across three
requirements**, not a refactor:

- Every existing receipt and every recorded log head becomes uninterpretable. There is no
  migration path that preserves an existing ordering log; it would have to be re-anchored.
- The ledger and the guest must agree on the new identity byte string. Today they agree
  trivially, because both use `signing_bytes()` from one crate.
- **There is no baseline to copy.** hcsshim has no ordering log — FR-1j is a kata superset
  — so unlike the rest of this note, the design cannot be taken from C-ACI. It has to be
  invented and justified.

A further wrinkle: COSE_Sign1 bytes are not canonical either. Two encoders can produce
different envelopes for the same logical content, so hashing "the envelope" reintroduces
exactly the canonicality problem the statement's re-encode check solves today — unless the
`Sig_structure` is used, which is canonical by construction because the signer had to
produce it to sign at all. That is the option to pursue.

## 5. Recommendation

Do not take this as a refactor, and do not take it opportunistically alongside unrelated
FR-1 work. It is worth doing when FR-1f/FR-1j are open for other reasons — at which point
the receipt/ordering identity should be re-based onto the `Sig_structure` and the statement
deleted in the same change.

Until then the statement is stable, CBOR-encoded, canonicality-checked and no longer
reachable from any host-describable path. The residual costs are F-150 (fixable
independently by minting our own layer media type) and the absence of drop-in C-ACI signer
interop (which needs a payload-construction step, not a reimplementation).
