# BIP Draft v2.1: Co-signer Spending Policies

Original v2: October 5, 2026 · @Luke Carriere

v2.1, November 8, 2026, revising v2 (`468fdd1`). The document revision is v2.1; policy format version 0 and profile suffixes `-v0` are separate version identifiers. Canonical serialization and the complete wire schema remain open; this is not an implementation-ready interoperability specification.

This draft standardizes the policy a co-signer enforces, not the co-signer itself. One policy format works whether the co-signer sees the transaction, signs blind against a zero-knowledge proof, or runs in a secure enclave, and whether its conditions come from the transaction or from external oracles.

## Preamble

```
BIP: ?
Layer: Applications
Title: Co-signer Spending Policies
Authors: Luke Carriere <luke@lukecarriere.com>
Status: Draft
Type: Specification
License: BSD-2-Clause
Discussion: https://mirror.b10c.me/lists/bitcoindev/CAFYm+Pe8_a568_=GtHhHMVZUF8j8JSS_eYJEaTSbJnOn-dkCHw@mail.gmail.com/
            https://delvingbitcoin.org/t/bip-draft-co-signer-spending-policies/2955
Requires: 89, 174, 327, 340, 341, 379, 380, 386
```

## Abstract

This document specifies a portable spending policy for outputs that include one or more policy-enforcing co-signers. A policy defines the allowed spend templates and the conditions for each, which can be checked against the transaction itself or against attestations from external oracles. It also defines how a group of co-signers is formed, and which spending paths stay available without them. Verification modes, evidence formats and accountability schemes are pluggable profiles. Co-signers must accept DLC oracle attestations, so existing oracles can serve as evidence sources unchanged.

## Motivation

A co-signer that signs only when a predicate holds is a well-known way to emulate covenants on today's Bitcoin. Towns observed in 2021 that multisig alone can enforce an off-chain covenant this way. Recent work has made such co-signers far more private and accountable:

- Halseth's blinded co-signers sign through blinded MuSig2, after checking a zero-knowledge proof that the spend follows a committed policy.
- Chain Code Delegation (BIP 89) stops a collaborative custodian from seeing the chain activity of keys it co-signs for.
- Predicate blind signatures let a signer blind-sign a transaction only if a proof shows it meets the signer's conditions.
- Rubin's Un-FE'd Covenants proposes oracle-assisted covenant emulation using off-chain computation, key management and BitVM-style economic incentives, without a soft fork. It is related work, not an accountability scheme specified or implemented by this draft.

Product documentation illustrates differing lending architectures, not adoption of this proposal. Lendasat's P2P documentation describes borrower/lender/Lendasat 2-of-3 escrow; it does not establish the original draft's claim of DLC-oracle settlement. Liquidium's current .fi documentation describes Internet Computer pool canisters and chain-key threshold signing, not that same 2-of-3 architecture. These are first-party product descriptions, not independently audited implementation or security findings; neither product is a confirmed CSP adopter.

These designs have different policy and trust interfaces; this draft does not establish that every pair is incompatible. The proposed interoperability gap is a shared description of what may be signed and when. This proposal standardizes that description and leaves the mechanisms to profiles, so existing and future designs can implement the same policy.

## Design principles

1. **Script first.** Anything Bitcoin script can enforce MUST be enforced there. Co-signers enforce only what script cannot.
2. **Never stuck.** Every policy keeps at least one spending path that needs no co-signer.
3. **Policy, not mechanism.** How a co-signer verifies, and what backs its honesty, are profiles chosen per policy.
4. **Reuse before inventing.** Use descriptors, miniscript, PSBTs, MuSig2, BIP 89 and DLC attestations as they exist.
5. **Honest about trust.** Conditions enforced only by co-signers are emulated covenants, and are only as strong as the co-signer group.

## Specification

The key words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

### Overview

A policy has five parts:

| Part | Defines |
| --- | --- |
| Output | The Taproot descriptor holding the funds, including co-signer and fallback paths |
| Co-signer group | Which co-signers take part, and how many must sign |
| Templates | The transactions the co-signers may sign, with exact outputs |
| Conditions | What must be true before each template can be signed |
| Profiles | The verification mode, evidence formats and optional accountability scheme |

### Policy validity and commitment

The policy ID uses the BIP340 tagged-hash construction with tag `CSP/policy/v0`:

```
policy_preimage = encode_format_version(format_version) || canonical_policy_body(policy)
policy_id = tagged_hash("CSP/policy/v0", policy_preimage)
tagged_hash(tag, message) = SHA256(SHA256(UTF8(tag)) || SHA256(UTF8(tag)) || message)
```

The format version MUST be the first field of the application-message preimage (`policy_preimage`, after the tagged-hash domain prefix). The version is serialized once; `canonical_policy_body` serializes every other policy field. It MUST commit to the complete verifier semantics, including selected profile names/versions and every evidence condition, not merely to a parseable document. The exact encoding/framing of the version and body is **TBD pending canonical JSON versus TLV**. This notation specifies ordering and domain separation, not final wire bytes. If JSON is chosen, its canonicalization rule must be reconciled explicitly with the version-first requirement; ordinary JSON object insertion order is not a specification.

For the selected format and profile versions, at every nesting level:

- Unknown fields make the policy invalid. A co-signer **MUST refuse** to sign under it; fields MUST NOT be ignored. Extensions require new profile/format versions with defined semantics, not silently accepted fields.
- Duplicate keys make the policy invalid, even if the repeated values are equal. A parser MUST detect duplicates before constructing a map that could overwrite them. The selected TLV mapping, if used, MUST likewise define and reject duplicate singleton fields.
- There are **no implicit defaults**. Every field defined by the selected schema/profile MUST be serialized explicitly, including `format_version` and each evidence condition's `require_final`. A missing field makes the policy invalid. How explicitly empty/not-applicable fields are represented is part of the still-pending schema, not permission to omit them.
- Unsupported versions or profiles, malformed fields and invalid policies MUST NOT be interpreted permissively or substituted with a different version's semantics. Validation MUST precede signing and policy-ID acceptance.

These fail-closed rules apply to the CSP policy document, not to unrelated fields of the enclosing PSBT; existing PSBT extension rules remain unchanged.

Policy and statement commitments use distinct CSP domains: `CSP/policy/v0` and `CSP/statement/v0`. Every CSP application tag MUST be distinct from every other CSP application tag and from the BIP340/BIP341 tags, including `BIP0340/aux`, `BIP0340/nonce`, `BIP0340/challenge`, `TapLeaf`, `TapBranch`, `TapTweak` and `TapSighash`. This does not modify the internal tags used by the BIP340 signature algorithm or BIP341 transaction signing.

### Output

- The output MUST be a Taproot output described by a `tr()` descriptor (BIP 386), with miniscript leaves (BIP 379).
- Each template MUST correspond to a script leaf that requires the co-signer group plus the participants named for that template.
- Any condition expressible in script, such as an absolute or relative timelock, MUST be placed in that template's leaf rather than left to the co-signers.
- The output MUST include at least one path that requires no co-signer, such as all participants together or a participant after a long timelock.

### Co-signer group

The policy MUST define the co-signer group in one of two forms:

- **Aggregate:** n co-signers combined into one MuSig2 key (BIP 327). Every co-signer must sign, so one honest co-signer can block an invalid spend, but one offline co-signer blocks a valid one.
- **Threshold:** k of n co-signer keys in the leaf script (`multi_a`). Any k can sign, which tolerates offline co-signers. A k-of-n group blocks an invalid spend as long as fewer than k co-signers are willing to sign it, i.e. n−k+1 honest refusals suffice. For example, 2-of-4 needs 3 honest refusals.

Co-signer keys SHOULD be supplied under Chain Code Delegation (BIP 89), so a co-signer cannot scan the chain for other outputs under the same policy or key.

### Templates

A template MUST define every output exactly. Output amounts are expressions built from:

- literal values and named policy variables,
- values carried by evidence (for example, an attested price),
- `+`, `−`, `×`, `÷`, `min` and `max`, with a rounding direction stated for each division,
- `remainder`: the input total minus all other outputs and the fee.

A template MUST have at most one `remainder` output. The fee is set by the participants; co-signers check every non-fee output.

### Conditions

A condition combines the following with AND and OR:

- **Transaction predicates:** properties of the spending transaction, such as its outputs matching the template. These are checked by every verification mode.
- **Evidence conditions:** k of n named attestors attest an event outcome, or attested numeric values compare to a threshold.
- **Time conditions:** only where script can't express them. If script can, the condition belongs in the leaf (see Output).

Each evidence condition MUST name its attestors, the evidence profile, the event or event pattern, the threshold k, a maximum evidence age, and an explicitly serialized Boolean `require_final`. There is no default for `require_final`. If true, every attestation counted toward that condition's threshold MUST satisfy the selected profile's finality rule, in addition to the event/outcome, signature, threshold and age checks. If false, provisional statements may count; that trade-off MUST be explicit in the policy. Missing or malformed finality information MUST NOT be treated as final.

### Profiles

A policy names one profile of each kind. Profiles are identified by name and version, and can be specified in separate documents.

**Verification modes**

| Mode | Co-signer sees | Status |
| --- | --- | --- |
| `open-v0` | The policy, the PSBT and the evidence | Specified here |
| `blinded-v0` | A blinded signing request plus a zero-knowledge proof that it satisfies the policy committed by the policy ID | To be specified, building on blinded MuSig2 and predicate blind signatures |
| `enclave-v0` | Everything, inside a secure enclave whose code is attested remotely | To be specified |

**Evidence profiles**

- `dlc-v0`: oracle announcements and attestations as defined in the DLC specifications, verified exactly as those specifications require. All co-signers MUST support it.
- `statement-v0`: a fallback for attestors that don't pre-commit nonces. A BIP340 signature over the tagged hash, with tag `CSP/statement/v0`, of the ordered tuple `(event_id, outcome, timestamp, final)`. The Boolean `final` is signed, MUST be explicitly present (true or false), and has no implicit default. It states whether the attestor declares that outcome final for that event; it does not prove the outcome is true. If a condition has `require_final=true`, only valid statements with signed `final=true` may count. The tuple's exact byte encoding/framing is **TBD** with the serialization decision and must have vectors before interoperability can be claimed. The authors would prefer to move this format into the DLC specifications if their maintainers agree.

For `dlc-v0`, a valid attestation matching the event announcement and pre-committed nonce(s) is treated as effectively final for `require_final`: conflicting outcomes signed using the same event nonce expose the oracle's key (for digit decompositions, at a conflicting signed digit). This is the DLC nonce-precommitment security property, not a new flag or change to DLC messages. It does not guarantee truthful observations or prevent oracle disappearance; a changed announcement or different nonce MUST NOT be substituted for the one bound by the policy's event condition.

**Accountability (optional)**

A policy MAY name an accountability scheme backing the co-signers, such as bonds that can be taken by fraud proof. No scheme is specified here.

### Co-signer behavior in `open-v0`

A co-signer MUST sign a PSBT only if all of the following hold:

0. The policy is valid under its declared format and profile versions, satisfies the fail-closed rules above, and has the expected policy ID.
1. Every input is an output under this policy, spent through the leaf for the requested template.
2. Every output matches that template exactly, with all expressions evaluated.
3. The template's condition is met by valid evidence within its maximum age, including the profile-specific finality check whenever `require_final=true`.

Otherwise it MUST refuse. For each signature, it MUST produce a record with the policy ID, transaction ID, template and evidence used, and MUST provide it to every participant.

## Examples

### Collateralized loan

Collateral sits in a Taproot output with these paths:

| Path | Requires | Condition |
| --- | --- | --- |
| Key path | Borrower and lender | None: any cooperative spend |
| release | Borrower and co-signers | 2 of 3 attestors attest `paid` for the repayment event |
| liquidate | Lender and co-signers | 2 of 3 DLC price attestations show the loan's value threshold is crossed, or the term has ended and the repayment event is `not_paid` |
| recovery | Borrower alone | A long timelock in script, as a last resort if lender and co-signers vanish |

The liquidate template pays the lender BTC worth the amount owed at the mean attested price, with `remainder` to the borrower. Price evidence comes from existing DLC oracles, unchanged.

### Vault

The same format describes a blinded co-signer vault. The unvault template's relative timelock is in its leaf, so script enforces it. Using `blinded-v0`, the co-signers check only that the final spend matches the template, without seeing the transaction.

### Escrow

Templates `pay_seller` and `refund_buyer`, each conditioned on an attested delivery outcome, with a buyer-and-seller key path for cooperative settlement. In the `statement-v0` version of this example, each evidence condition explicitly sets `require_final=true`: a recent but provisional delivery statement MUST NOT authorize release or refund.

If an attestor never declares finality, an alternative template can explicitly set `require_final=false` and wait out a defined dispute window before accepting a provisional outcome. That is a different policy, not an automatic exception to `require_final=true`. The policy must define the window's start/end and how corrections or conflicting statements are handled; those details are application-specific and unresolved here. Any enforceable deadline belongs in the script leaf under the script-first rule. This alternative does not create finality by waiting or prove that an unpublished correction does not exist; a co-signer-free cooperative or timelocked recovery path still addresses nonresponse. The example does not specify a complete dispute protocol.

## Rationale

- **Why standardize the policy, not the co-signer?** Co-signer designs are improving quickly: blinding, delegation, enclaves, bonds. A shared policy lets each improvement apply to every application without changing it.
- **Why script first?** A condition left to co-signers is only as strong as they are. Concept reviews of earlier co-signer vaults showed timers enforced off-chain giving weaker guarantees than they appeared to.
- **Why a mandatory fallback path?** Co-signers must be online to sign. Without a path that bypasses them, an outage freezes funds.
- **Why both aggregate and threshold groups?** They trade safety against liveness in opposite directions, and different applications need different balances.
- **Why DLC attestations?** DLC oracles already publish BIP340-based price and event attestations. Requiring that format makes every existing oracle a possible evidence source.
- **Why separate attestors from co-signers?** An escrow agent that both decides the outcome and signs holds too much power. Here, attestors state facts and co-signers only apply rules.

## Forward compatibility

Templates whose outputs don't depend on evidence describe exact transactions. If a template covenant such as OP\_CHECKTEMPLATEVERIFY is ever activated, such templates could move into script and no longer need co-signers. This proposal takes no position on any soft fork.

## Security considerations

- **Trust model.** Conditions not in script are emulated covenants. With an aggregate group they hold if one co-signer is honest; a k-of-n group blocks an invalid spend as long as fewer than k co-signers are willing to sign it, i.e. n−k+1 honest refusals suffice.
- **Liveness.** Co-signers must be online. The mandatory fallback path limits the damage of an outage, but delays settlement.
- **Collusion.** Co-signers plus one participant can spend through that participant's templates. Accountability profiles and public records reduce this risk without removing it.
- **Evidence.** k colluding attestors can satisfy a condition falsely. Policies should use independent attestors and k ≥ 2. Maximum age and event binding restrict replay but do not establish truth or finality. Signed `final=true` is an attestor commitment, not proof of real-world correctness; a finality requirement can reduce liveness if the attestor never marks finality. Dispute windows need explicit correction/conflict rules.
- **Semantic agreement.** Policy-ID agreement is meaningful only if verifiers enforce the same versioned schema, explicit fields and rejection rules. Implementations MUST NOT discard unknown fields, overwrite duplicate keys, or inject defaults before deciding whether a policy is valid.
- **Privacy.** In `open-v0` co-signers see amounts and addresses. BIP 89 limits what they learn about other outputs, and `blinded-v0` limits what they learn about the spend itself.
- **Value thresholds.** Prices can move past a threshold between attestations. Policies should leave a margin.

## Backwards compatibility

No consensus change is required. Wallets that support Taproot descriptors, miniscript and PSBTs can hold policy outputs. DLC oracles need no changes to provide evidence.

## Reference implementation

Not yet available. An `open-v0` co-signer, test vectors and a policy validator will be published before this draft seeks Complete status.

## Test vectors

This section records the required rejection cases and exact-byte vector deliverables; it does **not** claim executable fixtures or hash vectors have already shipped.

### Rejected policy documents

Starting from a complete valid policy fixture `P` for the selected schema, independent implementations MUST reject each of these single mutations, MUST NOT sign under it, and MUST NOT silently normalize it into an accepted policy:

1. **Unknown field:** add an undefined field such as `unexpected_field` at the policy root or within a selected profile's object. Expected result: invalid policy / refusal.
2. **Duplicate key:** repeat a field such as `format_version` in the serialized document, even with the same value. Expected result: invalid policy / refusal before map normalization. A JSON fragment such as `"format_version":0,"format_version":0` illustrates the defect, not a complete policy or a canonical encoding. If TLV is selected, include duplicate-singleton-type fixtures.
3. **Missing field:** remove `format_version`, or remove a required field such as an evidence condition's `require_final`. Expected result: invalid policy / refusal; no inferred default.

Complete rejected documents and their base fixture are **TBD**, pending the complete schema and JSON/TLV decision. These mutation descriptions are semantic test requirements, not byte-level conformance vectors.

### Exact policy-ID bytes

**TBD pending canonical JSON versus TLV. No policy-ID hash values are asserted here.** For every accepted fixture, publish the complete document, declared version/profile identifiers, exact version encoding, exact canonical body bytes (hex), complete `policy_preimage` bytes (hex), UTF-8 tag bytes and expected 32-byte policy ID (hex). Include explicit defaults/empty values, nested fields, evidence finality choices and any canonicalization edge cases. Two independent implementations must reproduce the same bytes and IDs. Publish exact rejected-document bytes for the cases above as well; rejection must not yield an accepted policy ID.

### Evidence finality cases

Include signed `statement-v0` fixtures for `final=false` and `final=true`; a missing flag; a tampered flag; and both values of `require_final`. Provisional, missing or tampered flags MUST NOT satisfy `require_final=true`; a correctly signed final statement still must pass event/outcome, age and threshold checks. Include a valid DLC announcement/attestation pair showing effective finality, plus mismatched-event/nonce rejection. Exact statement bytes, signatures and DLC fixtures remain **TBD**; no cryptographic values are invented in this review revision.

## Open questions for discussion

- [ ] Serialization: canonical JSON for readability, or TLV to match the DLC specifications?
- [ ] Can `blinded-v0` be specified now from blinded MuSig2 and predicate blind signatures, or should it wait for more implementation experience?
- [ ] Should `statement-v0` live here or in the DLC specifications?
- [ ] Is the output expression language the right size, or can an existing format replace it?
- [ ] Should policies also cover outputs on layers such as Ark? Ecosystem commentary is not evidence that a particular product implements CSP or that this draft specifies Ark policy enforcement.
- [ ] Should the fallback path be required to be timelocked, so it can't be used to bypass co-signers early?

## Related work

Dates below are publication dates/months verified from opened sources. First-party product documentation and ecosystem commentary are attributed descriptions, not independent implementation, safety or legal verification.

- Anthony Towns, [multisig as an off-chain recursive covenant](https://mirror.b10c.me/lists/bitcoindev/20210705050421.GA31145@erisian.com.au/), bitcoin-dev, July 2021. This is the direct conditional-co-signing precedent cited here.
- Johan Halseth, [Building a vault using blinded co-signers](https://delvingbitcoin.org/t/building-a-vault-using-blinded-co-signers/2141), Delving Bitcoin, December 2025 ([Optech summary](https://bitcoinops.org/en/newsletters/2026/01/02/), January 2, 2026). The published vault is a prototype, not evidence of production-safe CSP.
- Jurvis Tan and Jesse Posner, [Chain Code Delegation: Private Access Control for Bitcoin Keys](https://delvingbitcoin.org/t/chain-code-delegation-private-access-control-for-bitcoin-keys/1837), Delving Bitcoin, July 2025, and [BIP 89](https://bips.dev/89/). BIP 89's canonical repository reports **Deployed**, version 1.0.0, checked November 8, 2026 (CST); this is a status check, not a security audit or endorsement.
- Georg Fuchsbauer and Mathias Wolf, [Concurrently Secure Blind Schnorr Signatures](https://repositum.tuwien.at/handle/20.500.12708/200888), EUROCRYPT 2024, published May 8, 2024, pp. 124–160. Includes predicate blind signatures and Bitcoin transaction-condition applications. This corrects the v2 citation's descriptive rather than actual title.
- Jeremy Rubin, [Un-FE'd Covenants](https://rubin.io/bitcoin/2024/11/26/unfed-covenants/), author's publication, November 26, 2024. Oracle-assisted covenant emulation and BitVM-style incentives; not a CSP implementation.
- [B-SSL concept review](https://delvingbitcoin.org/t/concept-review-b-ssl-bitcoin-secure-signing-layer-covenant-free-vault-model-using-taproot-csv-and-cltv/2047), Delving Bitcoin, October 2025. Relevant to distinguishing script-enforced delays from off-chain signer promises.
- [DLC specifications: oracle messages](https://github.com/discreetlogcontracts/dlcspecs/blob/master/Oracle.md). [NIP-88: DLC oracle announcement/attestation event kinds](https://github.com/nostr-protocol/nips/pull/1681), proposed by conduition in January 2025, is a **closed historical PR**, not an adopted NIP or normative CSP dependency.
- Giulio Caldarelli, [From Multi-sig to DLCs: Modern Oracle Designs on Bitcoin](https://arxiv.org/abs/2602.09822), arXiv v1, February 10, 2026. The metadata labels it **not peer reviewed**; cite as a survey/preprint, not independent verification of every claim it discusses.
- Product descriptions: [Liquidium security, governance & risk](https://liquidium.fi/docs/technical/security/governance-and-risk), current pool-canister/chain-key architecture; [Lendasat P2P loan protocol](https://lendasat.com/docs/p2p-loans/concepts/protocol), documented 2-of-3 escrow. Neither source establishes the original sweeping claim that both products use the same 2-of-3 plus DLC-oracle settlement construction. Product operation/security claims remain unaudited here.
- [Building Financial Infrastructure on Bitcoin with Ark](https://blog.bitfinex.com/industry-news/building-financial-infrastructure-on-bitcoin-ark/), Bitfinex, July 24, 2026. Ecosystem commentary, not primary proof of another team's product or CSP deployment.
- Robin Linus (@RobinLinus), [Perpetually KYC'd Coins Using Evil Covenants](https://delvingbitcoin.org/t/perpetually-kycd-coins-using-evil-covenants/556), Delving Bitcoin, February 2024. Adversarial covenant/whitelist discussion, including comparisons with multisig co-signers; not a recommendation to implement its proposed opcodes or legal claims.
- Anthony Towns, [Soft-forks and schnorr signature aggregation](https://mirror.b10c.me/lists/bitcoindev/20180321040618.GA4494@erisian.com.au/), bitcoin-dev, March 2018. Historical discussion of aggregation, covenant checks and additional sign-off, not an already-deployed CSP mechanism.
- @evd0kim, [Institutional Grade Spending Policies for Statechains](https://delvingbitcoin.org/t/institutional-grade-spending-policies-for-statechains/2874), Delving Bitcoin, September 2026. BLISK proposal combining Boolean approval policies with blind MuSig2 statechain transfers; not a verified CSP implementation or audited cryptographic construction.
- @josh, [Post-Signature Cross-Input Scripting Using the Taproot Annex](https://delvingbitcoin.org/t/post-signature-cross-input-scripting-using-the-taproot-annex/1520), Delving Bitcoin, March 2025. Includes signature-time extra co-signing conditions and explicitly requires a soft fork; unlike CSP, it is not presented as functionality available under current consensus rules.
- [BIP 174](https://bips.dev/174/) (PSBT), [BIP 327](https://bips.dev/327/) (MuSig2), [BIP 340](https://bips.dev/340/) (Schnorr), [BIP 341](https://bips.dev/341/) (Taproot), [BIP 379](https://bips.dev/379/) (Miniscript), [BIP 380](https://bips.dev/380/) (descriptors), and [BIP 386](https://bips.dev/386/) (`tr()` descriptors).

## Changelog

### v2 to v2.1 (November 8, 2026)

- Credit **@tehiyadu** for requiring the policy commitment to bind exact verifier semantics: reject unknown fields, duplicate keys and missing fields; eliminate implicit defaults; put the format version first; and version/domain-separate policy commitments with `CSP/policy/v0`. Clarify separation from BIP340/BIP341 domains. These address Luke's October 28 Delving reply 4.
- Credit **@PREDGE** for identifying provisional/corrected outcome risk and requesting exact policy-ID bytes: sign statement finality, permit evidence conditions to require it, require final statements for the escrow example, and explain the dispute-window alternative and DLC nonce-precommitment property. These address Luke's October 28 reply 5.
- Correct the k-of-n safety condition in Co-signer group and Security: fewer than k co-signers willing to sign an invalid spend, equivalently n−k+1 honest refusals, block it. Add the 2-of-4 example (3 honest refusals).
- Add rejected-document requirements and an exact-byte vector plan, with all wire bytes, signatures and policy IDs explicitly TBD until the schema/serialization is settled. No executable vectors or reference implementation are claimed.
- Correct or qualify Related work citations and unsupported product claims; add four verified historical links. Record BIP 89's actual Deployed status; add 89, 327 and 386 to Requires; fill Discussion with the verified announcement and Delving URLs.
- Changing the policy hash tag/preimage and statement tuple is incompatible with v2 policy IDs/statements; never silently reinterpret old material. The proposed `-v0` profile labels are retained for review because they were still unimplemented draft profiles, not established compatibility guarantees. Profile naming/version migration needs an explicit decision before interoperability claims.

## Copyright

This document is licensed under the BSD-2-Clause license.
