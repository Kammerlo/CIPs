---
CPS: "?"
Title: Discoverability and machine-readable description of programmable token substandards
Category: Tokens
Status: Open
Authors:
    - Thomas Kammerlocher <thomas.kammerlocher@cardanofoundation.org>
Proposed Solutions: []
Discussions:
    - CIP-0113? | Programmable token-like assets (candidate): https://github.com/cardano-foundation/CIPs/pull/444
    - CPS-0003 | Smart Tokens: https://github.com/cardano-foundation/CIPs/tree/master/CPS-0003
Created: 2026-08-20
License: CC-BY-4.0
---

## Abstract

[CIP-0113][CIP-0113] (candidate) specifies programmable tokens as a layered system: a shared on-chain registry and base-layer scripts (Layers 1 and 2) whose transaction shape is fully specified, and a per-token *substandard* (Layer 3) whose transfer, third-party and issuance logic is user-defined. The base layer is precise about what every transfer must contain; the substandard layer is, by design, open.

This openness is the point of programmable tokens, but it leaves downstream tooling — wallets, DEXs, lending protocols, explorers, indexers, hardware-wallet firmware, auditors — without a way to (1) learn *which* substandard a given token policy belongs to and (2) learn *how* a valid transaction for that substandard must be built: which redeemer, which reference inputs, which user- or global-state UTxOs, which additional proofs, which output datums. Today this knowledge exists only in each substandard's source repository and in hand-written integrations. Every new substandard requires every wallet and dApp to integrate it by hand, and the result does not scale beyond a handful of implementations.

This CPS asks for a standard way to identify a token's substandard on-chain, a registry of known substandards, and a machine-readable description of the transactions a substandard accepts, so that generic tooling can build and explain programmable-token transactions without per-substandard code.

## Problem

### What CIP-0113 specifies, and what it leaves open

CIP-0113 states that *"the term 'substandard' indicates a specific implementation of all the token-specific components (Layer 3) that guarantees a certain behaviour and a consistent way to build transactions. Any programmable token MUST belong to a certain substandard, otherwise wallets and dApps don't know how to properly interact with them."*

It then specifies Layers 1 and 2 in detail — the `RegistryNode` datum, the `TransferAct` / `ThirdPartyAct` redeemers, `RegistryProof` ordering rules, the withdraw-zero delegation pattern, smart-wallet address derivation — and gives a complete example of a native transfer. In that example, the substandard-specific part is a single opaque value (`substandardTransferRedeemer`), and the specification's own wording for substandard-specific reference inputs is *"the exact requirements depend on the substandard"*.

Three concrete gaps follow:

1. **No on-chain substandard identity.** The `RegistryNode` datum contains the token policy, the linked-list `next` pointer, the credentials of the transfer and third-party scripts, and an optional global-state policy. It contains no field that names the substandard, its version, or a pointer to a description. The only way to infer "this is a freeze-and-seize token" is to already know the script hashes of that substandard and compare them — which fails as soon as the scripts are parameterised per token, which they usually are.

2. **No registry of substandards.** The list of existing substandards is a hand-maintained table in the CIP text (three rows at the time of writing: a dummy token, a freeze-and-seize stablecoin, and a regulatory security-token standard), each pointing at a source repository. Nothing ties a repository to the on-chain scripts a token actually uses, and nothing tells a wallet that a newly registered policy belongs to a substandard it already supports.

3. **No machine-readable description of substandard transactions.** A substandard's transfer may require, beyond the CIP-0113 base shape: a specific redeemer constructor and fields; reference inputs for sender and/or receiver state UTxOs located by NFT; a global-state reference input; additional withdraw-zero scripts; Merkle or signature proofs obtained from an off-chain service; specific output datums; or specific ordering of inputs and outputs. [CIP-0057][CIP-0057] blueprints describe the datum and redeemer *schemas* of each validator, which is necessary but not sufficient — a blueprint does not say which reference inputs to include, how to find them, or what the resulting transaction must look like as a whole.

### Why this is a problem now

Programmable tokens are only useful if ordinary wallets and dApps can move them. CIP-0113 itself lists *"a widely adopted wallet that can read and display programmable token balances to users and allow the user to conduct transfers of such tokens"* as an acceptance criterion. Experience from early integrations shows where the cost lands:

- A wallet integrating CIP-0113 can implement balance display and transaction history generically, because those depend only on Layers 1 and 2. It cannot generically *build* a transfer, because the substandard redeemer and reference inputs are unknown. The practical outcome is a wallet that will co-sign a substandard-aware dApp's transaction but refuses to construct one itself — which shifts the whole burden onto every token issuer to ship their own dApp.
- Even substandards that are structurally similar (e.g. two compliance tokens) differ in details that matter for transaction building: which party must present state, whether state is one UTxO per user or one global UTxO, how proofs are supplied, whether the first mint may be combined with registration, and how admin actions are authorised. These differences are invisible on-chain.
- A DeFi protocol deciding whether to accept a programmable token as collateral needs to know what the third-party script permits (freeze, seize, forced transfer, clawback). CIP-0113 tells protocols to *"review `thirdPartyTransferLogicScript` to understand admin capabilities"* — i.e. read the source. There is no declared capability set to check against.
- Substandards evolve. Several implementations make the transfer or issuance logic upgradeable (rotatable authority scripts, registry-node updates), so the script hashes a wallet matched against yesterday may not be the ones in force tomorrow. Without a versioned description, tooling silently builds invalid transactions.

### Relationship to CPS-0003

[CPS-0003][CPS-0003] asks whether Cardano should have smart tokens at all and lists among its open questions *"How would wallets know how to interact with these tokens? — smart contract registry?"*. CIP-0113 answers the first question and provides a registry *of tokens*. This CPS is the next level down: assuming CIP-0113 (or a successor) exists, how do tools learn to interact with the user-defined part of each token? It is scoped to the substandard layer and addressed to tooling builders; it is not a re-statement of CPS-0003.

## Use Cases

**Wallet building a transfer of an unfamiliar token.** A user holds a programmable token issued last week by a substandard the wallet team has never seen. The user wants to send some to a friend. Today the wallet can show the balance but must refuse to build the transfer, or hand the user to an issuer-specific web app. The wallet should be able to resolve the token's substandard, obtain a description, and construct the transaction — or at least tell the user precisely why it cannot.

**Wallet consent screen.** A dApp presents a programmable-token transaction for signature. The wallet can see a withdraw-zero from a script it does not know and a redeemer it cannot decode. To show a meaningful "you are transferring X to Y under rules Z" screen — or, for hardware wallets, to decide whether the transaction is safe to blind-sign — it needs to know what the substandard scripts and redeemer mean.

**DEX or lending protocol integration.** A protocol wants to list a programmable token. It needs to know, before integration and in an automated way, what the substandard requires in every swap or liquidation transaction and which admin powers exist over the token (can collateral be frozen or seized?). Today this is a manual code review per token.

**Explorer / indexer.** An explorer wants to label a policy as "CIP-0113, substandard S, version V" and render its transactions with the substandard's vocabulary (freeze, seize, KYC-gated transfer). Nothing on-chain supports that classification.

**Issuer of a new substandard.** A team builds a compliance token for a regulated market. Their goal is for existing wallets to support it without the team writing wallet plugins. Today, adoption requires convincing each wallet vendor to integrate their specific scripts.

**Auditor / regulator.** Someone needs to verify that the transaction-building rules a wallet implements for a substandard correspond to the scripts actually deployed on-chain, and that an issuer's claimed capabilities ("no seizure") match the deployed third-party script.

In all cases the current alternative is *read the substandard's source repository and hand-write the integration*, which is unsuitable because it does not scale with the number of substandards, breaks silently on upgrades, and cannot be verified against the chain by the tool that relies on it.

## Goals

Ranked by importance:

1. **On-chain substandard identity.** Given a registered programmable-token policy, tooling must be able to determine, from on-chain data alone, which substandard (and which version of it) the token belongs to, and to locate its description. The identity must be bound to the scripts actually in force — not merely asserted in free text — so that a description cannot be attached to scripts it does not describe.

2. **Machine-readable transaction description.** A substandard must be able to publish a description from which generic tooling can construct valid transactions for each operation it supports (at minimum: user transfer; ideally also third-party actions and issuance). The description must cover the full transaction shape required by the substandard on top of the CIP-0113 base layer — redeemer constructors and fields, required reference inputs and how to locate them, required withdrawals, required output datums, ordering constraints — and must be precise enough to be executed, not just read.

3. **Resolution of dynamic inputs.** Many substandards need inputs that cannot be derived from the chain by pattern alone: a proof from an off-chain attestation service, a Merkle inclusion proof against a published root, a state UTxO found by NFT. The description must be able to declare such inputs, their provenance, and how to obtain them, without the description itself becoming executable code that tooling must trust.

4. **Human-readable semantics.** Alongside the executable shape, a description must carry what each operation and redeemer *means* (e.g. "transfer subject to receiver allowlist", "admin seizure"), so wallets can render consent screens and protocols can reason about capabilities. Capability declarations (can freeze, can seize, can force-transfer, can pause, can clawback) should be expressible in a way that tools can query.

5. **Registry and versioning.** There must be a discoverable registry of substandards and their descriptions, with versioning that survives upgradeable substandards (rotated authority scripts, registry-node updates) and deployments of CIP-0113 on multiple networks or framework versions.

6. **Minimal trust and verifiability.** A tool must be able to check that a description corresponds to the on-chain scripts (e.g. by script hash or by blueprint hash) rather than trusting its source. Descriptions must not be able to cause a tool to build a transaction that does something other than what the description claims.

Non-goals:

- Changing how Layers 1 and 2 of CIP-0113 validate transactions.
- Mandating a single off-chain SDK or language; the goal is a description that any SDK can consume.
- Standardising the *content* of substandards (which compliance rules a token should enforce). This CPS is about describing whatever a substandard does, not prescribing it.

## Open Questions

1. **Where does substandard identity live?** In a new `RegistryNode` field (changing the datum changes the registry scripts and therefore the CIP-0113 deployment), in transaction metadata of the registration transaction, in a separate on-chain registry keyed by script hash, or derived purely from script hashes? What is the migration path for tokens already registered under the current datum?

2. **What is the description format?** Should it extend CIP-0057 blueprints (which already describe datums and redeemers) with a transaction-shape layer, adopt an existing transaction-template language, or define a new schema? What is the minimal expressive power needed to cover existing substandards, and where does expressiveness become a security risk?

3. **How are dynamic inputs described?** How does a description declare "sender must include a reference input holding NFT X", "receiver's KYC proof from service Y", or "global-state UTxO identified by policy Z" in a way that is precise, verifiable, and not arbitrary code? How are off-chain services that supply proofs identified and trusted?

4. **How is a description bound to the scripts?** By per-script hash, by hash of the full CIP-0057 blueprint, by the parameterised script template plus parameters, or by signature of a known issuer? How does binding survive upgradeable substandards whose script hashes rotate?

5. **Who maintains the registry, and how are entries trusted?** Permissionless on-chain registration with hash binding, a curated off-chain list, or both? How does a tool distinguish an official description from a malicious one that claims to describe a popular substandard?

6. **How should capabilities be declared and checked?** Can a capability set (freeze, seize, pause, force-transfer, mint cap, …) be derived or at least cross-checked from the scripts, or is it necessarily an issuer claim?

7. **What must a wallet be able to do with a description alone?** Is the target "build a transfer", "build any user-initiated operation", or also "render any substandard transaction for consent"? What is the minimum a hardware wallet needs?

8. **How does this interact with CIP-0113 versioning?** A CIP-0113 version is identified by the hash of its bootstrap transaction; a substandard may be deployed under several versions and networks. How does a description refer to the framework version it targets?

9. **Can part of this be enforced on-chain?** For example, can registration require a valid description pointer, or can a substandard's scripts commit to their own description hash?

## Copyright

This CPS is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).

[CIP-0113]: https://github.com/cardano-foundation/CIPs/pull/444
[CIP-0057]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0057
[CPS-0003]: https://github.com/cardano-foundation/CIPs/tree/master/CPS-0003
