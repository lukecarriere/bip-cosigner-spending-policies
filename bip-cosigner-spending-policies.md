# BIP Draft v2: Co-signer Spending Policies

Oct 5, 2026 · @Luke Carriere

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
Discussion: (to be added after posting)
Requires: 174, 340, 341, 379, 380
```

## Abstract

This document specifies a portable spending policy for outputs that include one or more policy-enforcing co-signers. A policy defines the allowed spend templates and the conditions for each, which can be checked against the transaction itself or against attestations from external oracles. It also defines how a group of co-signers is formed, and which spending paths stay available without them. Verification modes, evidence formats and accountability schemes are pluggable profiles. Co-signers must accept DLC oracle attestations, so existing oracles can serve as evidence sources unchanged.

## Motivation

A co-signer that signs only when a predicate holds is a well-known way to emulate covenants on today's Bitcoin. Towns observed in 2021 that multisig alone can enforce an off-chain covenant this way. Recent work has made such co-signers far more private and accountable:

- Halseth's blinded co-signers sign through blinded MuSig2, after checking a zero-knowledge proof that the spend follows a committed policy.
- Chain Code Delegation (BIP 89) stops a collaborative custodian from seeing the chain activity of keys it co-signs for.
- Predicate blind signatures let a signer blind-sign a transaction only if a proof shows it meets the signer's conditions.
- Rubin's Un-FE'd Covenants back co-signers with bonds that can be taken through BitVM fraud proofs if they sign against the rules.

Meanwhile, products already use policy co-signers in practice. Liquidium and Lendasat both pair a 2-of-3 multisig with oracle-driven settlement for bitcoin-backed loans.

Each of these defines its own policy, so a wallet built for one can't use a co-signer, oracle or verifier built for another. What's missing is a shared description of what may be signed and when. This proposal standardizes that description and leaves the mechanisms to profiles, so existing and future designs can implement the same policy.

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

The policy ID is the BIP340 tagged hash, with tag `CSP/policy`, of the policy's canonical serialization.

### Output

- The output MUST be a Taproot output described by a `tr()` descriptor (BIP 386), with miniscript leaves (BIP 379).
- Each template MUST correspond to a script leaf that requires the co-signer group plus the participants named for that template.
- Any condition expressible in script, such as an absolute or relative timelock, MUST be placed in that template's leaf rather than left to the co-signers.
- The output MUST include at least one path that requires no co-signer, such as all participants together or a participant after a long timelock.

### Co-signer group

The policy MUST define the co-signer group in one of two forms:

- **Aggregate:** n co-signers combined into one MuSig2 key (BIP 327). Every co-signer must sign, so one honest co-signer can block an invalid spend, but one offline co-signer blocks a valid one.
- **Threshold:** k of n co-signer keys in the leaf script (`multi_a`). Any k can sign, which tolerates offline co-signers but needs k honest ones to block an invalid spend.

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

Each evidence condition MUST name its attestors, the evidence profile, the event or event pattern, the threshold k, and a maximum evidence age.

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
- `statement-v0`: a fallback for attestors that don't pre-commit nonces. A BIP340 signature over the tagged hash, with tag `CSP/statement/v0`, of the event ID, outcome and timestamp. The authors would prefer to move this into the DLC specifications if their maintainers agree.

**Accountability (optional)**

A policy MAY name an accountability scheme backing the co-signers, such as bonds that can be taken by fraud proof. No scheme is specified here.

### Co-signer behavior in `open-v0`

A co-signer MUST sign a PSBT only if all of the following hold:

1. Every input is an output under this policy, spent through the leaf for the requested template.
2. Every output matches that template exactly, with all expressions evaluated.
3. The template's condition is met by valid evidence within its maximum age.

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

Templates `pay_seller` and `refund_buyer`, each conditioned on an attested delivery outcome, with a buyer-and-seller key path for cooperative settlement.

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

- **Trust model.** Conditions not in script are emulated covenants. With an aggregate group they hold if one co-signer is honest; with a threshold group, if k are.
- **Liveness.** Co-signers must be online. The mandatory fallback path limits the damage of an outage, but delays settlement.
- **Collusion.** Co-signers plus one participant can spend through that participant's templates. Accountability profiles and public records reduce this risk without removing it.
- **Evidence.** k colluding attestors can satisfy a condition falsely. Policies should use independent attestors and k ≥ 2. Maximum age and event binding prevent replay.
- **Privacy.** In `open-v0` co-signers see amounts and addresses. BIP 89 limits what they learn about other outputs, and `blinded-v0` limits what they learn about the spend itself.
- **Value thresholds.** Prices can move past a threshold between attestations. Policies should leave a margin.

## Backwards compatibility

No consensus change is required. Wallets that support Taproot descriptors, miniscript and PSBTs can hold policy outputs. DLC oracles need no changes to provide evidence.

## Reference implementation

Not yet available. An `open-v0` co-signer, test vectors and a policy validator will be published before this draft seeks Complete status.

## Open questions for discussion

- [ ] Serialization: canonical JSON for readability, or TLV to match the DLC specifications?
- [ ] Can `blinded-v0` be specified now from blinded MuSig2 and predicate blind signatures, or should it wait for more implementation experience?
- [ ] Should `statement-v0` live here or in the DLC specifications?
- [ ] Is the output expression language the right size, or can an existing format replace it?
- [ ] Should policies also cover outputs on layers such as Ark, where lending and escrow products are emerging?
- [ ] Should the fallback path be required to be timelocked, so it can't be used to bypass co-signers early?

## Related work

- Anthony Towns, [multisig as an off-chain recursive covenant](https://gnusha.org/pi/bitcoindev/20210705050421.GA31145@erisian.com.au), bitcoin-dev, July 2021
- Johan Halseth, [Building a vault using blinded co-signers](https://delvingbitcoin.org/t/building-a-vault-using-blinded-co-signers/2141), Delving Bitcoin, December 2025 ([Optech summary](https://bitcoinops.org/en/newsletters/2026/01/02/))
- [Chain Code Delegation: Private Access Control for Bitcoin Keys](https://delvingbitcoin.org/t/chain-code-delegation-private-access-control-for-bitcoin-keys/1837), Delving Bitcoin, July 2025, and [BIP 89](https://bips.dev/89/)
- [Predicate blind signatures for Schnorr](https://repositum.tuwien.at/handle/20.500.12708/200888), TU Wien
- Jeremy Rubin, [Un-FE'd Covenants](https://gnusha.org/pi/bitcoindev/30440182-3d70-48c5-a01d-fad3c1e8048en@googlegroups.com/T), bitcoin-dev, November 2024
- [B-SSL concept review](https://delvingbitcoin.org/t/concept-review-b-ssl-bitcoin-secure-signing-layer-covenant-free-vault-model-using-taproot-csv-and-cltv/2047), Delving Bitcoin, October 2025
- [DLC specifications: oracle messages](https://github.com/discreetlogcontracts/dlcspecs/blob/master/Oracle.md) and [NIP-88 (proposal)](https://github.com/nostr-protocol/nips/pull/1681)
- [From Multi-sig to DLCs: Modern Oracle Designs on Bitcoin](https://arxiv.org/pdf/2602.09822), 2026
- Products: [Liquidium](https://liquidium.fi/blog/secuirty), [Lendasat](https://solanacompass.com/projects/lendasat)
- [Building Financial Infrastructure on Bitcoin with Ark](https://blog.bitfinex.com/industry-news/building-financial-infrastructure-on-bitcoin-ark/), Bitfinex, 2026
- BIP 174 (PSBT), BIP 327 (MuSig2), BIP 340 (Schnorr), BIP 341 (Taproot), BIP 379 (Miniscript), BIP 380 and 386 (descriptors)

## Copyright

This document is licensed under the BSD-2-Clause license.
