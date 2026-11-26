---
CIP: "?"
Title: Web-Wallet Bridge - Encryption Keys
Category: Wallets
Status: Proposed
Authors:
    - Thomas Kammerlocher <thomas.kammerlocher@cardanofoundation.org>
    - Mateusz Czeladka <mateusz.czeladka@cardanofoundation.org>
Implementors:
    - Gero Wallet <https://gerowallet.io/>
Discussions:
    - Original PR: https://github.com/cardano-foundation/CIPs/pull/1309
Created: 2026-10-28
License: CC-BY-4.0
---

## Abstract

[CIP-30] lets a dApp ask a wallet to **sign** with the user's keys, but not to use a key for
**encryption**. Every asymmetric encryption scheme a dApp might use — encrypted amounts,
messages or metadata — needs the same step on the recipient's side: an
elliptic-curve Diffie–Hellman (ECDH) computation between a long-lived private key held by the
recipient and an ephemeral public key chosen by the sender.

This proposal defines a [CIP-30] extension that gives dApps that step without ever disclosing a
private key. A wallet derives, from its existing seed, a separate **encryption key for each
registered purpose**. With an explicit, separate and revocable permission per purpose, a dApp can
read the public key and ask the wallet to compute ECDH shared secrets against ephemeral keys
taken from specific on-chain references. The wallet exposes no export call and no general
decryption call. Purposes, curves and the encryption schemes that use a purpose are pluggable: this
CIP registers the **viewing** purpose with the **Jubjub** curve, for schemes whose proofs run over
BLS12-381, and defines how further purposes, curves and schemes are added.

## Motivation: Why is this CIP necessary?

### The missing primitive

Public-key encryption on elliptic curves follows one pattern (ECIES and its relatives):

1. The recipient holds a keypair `(sk, PK)` and publishes `PK`.
2. The sender picks an ephemeral scalar `e`, publishes `Epk = e·G` next to the ciphertext, and
   derives a symmetric key from the shared secret `S = e·PK`.
3. The recipient computes the same `S = sk·Epk` and decrypts.

Step 3 needs the recipient's private key. On Cardano that key should live in the user's wallet,
derived from the seed the user already backs up. [CIP-30] offers no way to use such a key, and
the alternatives are all unacceptable:

- **Exporting the private key to the dApp.** Any site that obtained it once could decrypt
  forever, and the user could neither see nor revoke that access.
- **Deriving a key from a signature** ("sign this message and we derive your key from it").
  Signatures are not secrets users know to protect, and a deterministic signature over a public
  constant can be requested again by any site.
- **Keeping the feature inside wallet UIs.** Every dApp would need wallet-specific integration,
  which defeats the purpose of a common bridge.
- **Reusing a signing key for encryption**, for example by converting the Ed25519 payment or stake
  key to X25519. One key then both signs and encrypts (see
  [A separate role](#a-separate-role-not-the-payment-or-stake-key)).
- **Managing keys outside the wallet.** The dApp then has to solve backup itself, and the user's
  seed no longer restores access.

### Prior attempts

The request is not new. [Issue #204] asked for `api.getSharedSecret` in 2022; it was closed in 2026
without a standard. On Ethereum, MetaMask deprecated `eth_getEncryptionPublicKey` and
`eth_decrypt`. As [ERC-5630] explains, the flaw was structural: one secret used for both signing and
encryption, and one hard-coded cipher inside the wallet. This proposal takes both lessons: a
separate key, and a wallet that performs key agreement only, leaving the scheme to the application.

### Use cases

- **Reading data encrypted to an account.** A scheme that hides values on chain, such as
  amounts, conveys each one to its recipient through ECDH with the recipient's **viewing key**,
  whether the scheme is built into the ledger or by a dApp. A dApp that displays such values, or
  opens outputs encrypted to the account, needs this step. Whether the viewing key also reveals
  values the account sent depends on the scheme. This purpose is specified here.
- **Confidential token amounts.** Confidential [CIP-113] programmable tokens can hide each amount on
  chain and seal it to the recipient's viewing key, with the token's issuer as a second reader of
  every amount. Such a scheme runs today on Preprod with a reference dApp and Gero Wallet's
  prototype of this extension.
- **Encrypted messages and metadata (candidate purpose).** [CIP-83] encrypts transaction
  messages with a shared passphrase, which both parties must agree out of band, or a publicly
  known default. Encrypting to the recipient's public key removes the shared secret. This CIP does
  not specify that purpose yet, but its design must accommodate it.

### Stakeholders

Wallet implementors, dApp developers building on encrypted data, issuers of confidential assets
who read their holders' amounts as a second reader, and the users who read data encrypted to their
accounts, or let a dApp they choose read it.

Following the precedent of [CIP-95], this is an additive, namespaced [CIP-30] extension and
changes no existing endpoint.

## Specification

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT" and "MAY" are to be interpreted as
described in [RFC 2119].

### Activation

A dApp requests this extension like any [CIP-30] extension:

```ts
const api = await cardano.{walletName}.enable({ extensions: [{ cip: ???? }] });
```

If the wallet enables it, `api.getExtensions()` MUST include `{ cip: ???? }` and the endpoints
below MUST be available under the namespace `api.cip????`.

> **Provisional number.** Until a CIP number is assigned, implementations use the provisional
> number `9001` in its place for activation and the namespace (the derivation tag keeps
> `cip????`): a dApp calls `enable({ extensions: [{ cip: 9001 }] })`; a wallet lists
> `{ cip: 9001 }` in `supportedExtensions`, reports it in `api.getExtensions()` when it enables it,
> and serves the endpoints under `api.cip9001`. The CIP editors have not assigned `9001`; it is far
> above the assigned numbers, so it collides with no published extension. This departs from
> [CIP-30]'s
> [Experimental API](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0030#experimental-api),
> which would place unnumbered endpoints under `api.experimental`, so that dApps and wallets
> activate the extension the same way before and after numbering. Like the `cip????` in the
> derivation tag, it changes once the CIP is numbered.

Enabling the extension grants **nothing** by itself. Every endpoint is also gated by a
[per-purpose permission](#permission).

### Concepts

- **Purpose:** what a key is used for, e.g. `viewing`. Each purpose has its own key,
  its own permission and its own rules (see [Purposes](#purposes)). Keys of different purposes
  are cryptographically independent, so granting one purpose never exposes another.
- **Algorithm profile:** the group in which the key lives, the encodings, and the validation
  rules for ephemeral keys (see [Algorithm Profiles](#algorithm-profiles)).
- **Scheme:** a concrete encryption format that uses a purpose's key: which outputs belong to it,
  where `Epk` sits, what makes an output the account's, and what is derived from the shared secret
  (see [Schemes](#schemes)). A purpose serves one or more registered schemes.
- **Generation:** a hardened index that versions a key, so it can be rotated.

### Data Types

#### HexString

A lowercase hexadecimal string with no `0x` prefix.

#### KeySelector

```ts
type KeySelector = {
  purpose: string;        // a registered purpose, e.g. "viewing"
  algorithm: string;      // an algorithm profile allowed for that purpose, e.g. "jubjub"
  accountIndex?: number;  // default 0
  generation?: number;    // default 0
};
```

`accountIndex` and `generation` MUST be integers in `[0, 2³¹)`.

#### PublicKeyInfo

```ts
type PublicKeyInfo = {
  purpose: string;
  algorithm: string;
  publicKeyHex: HexString; // encoded as defined by the algorithm profile
  accountIndex: number;
  generation: number;
  path: string;            // derivation path of the key material, e.g. "m/1852'/1815'/0'/6/0'"
  capabilities: Capabilities;
};

type Capabilities = {
  schemes: string[];       // registered scheme ids it scopes (and confirms, if confirming)
  scoping: boolean;        // answers only for outputs it resolved and checked (see Scoping)
  confirming: boolean;     // releases S only for outputs that open under S (see Confirming)
  secondReader: boolean;   // a scoping wallet that accepts outputs sealed to the account as a second reader
  maxBatch: number;        // the largest ecdhBatch it accepts: at least 512; 2^31 - 1 if it sets no cap
};
```

`capabilities` describes the wallet's behaviour for this purpose. It arrives with the public key, so
after the purpose's permission is granted and before the dApp's first `ecdh` call; a dApp uses it to
decide whether to pre-filter references, to expect `not-own-ref` after a prompt, and how to chunk.
`secondReader` implies `confirming` (see [Confirming](#purpose-viewing)), and `confirming` implies `scoping`.
`secondReader` is meaningful only with `scoping`: a wallet that does not scope answers for any `Epk` it
is given, second-reader outputs included, and reports `schemes: []`.

#### EcdhRequest

```ts
type EcdhRequest = {
  ephemeralPublicKeyHex: HexString; // Epk, exactly as published by the sender
  ref: string;                      // where Epk was published; format defined by the purpose
};
```

Each ephemeral key is always passed **together with a reference to where it was published**.
This lets the wallet scope its answers (see [Purposes](#purposes)) and makes requests auditable.

### Key Derivation

All encryption keys derive from one new [CIP-1852] role. Role `6` is reserved for the keys of this
extension and for the keys of names other CIPs register under it (below), in the same way
[CIP-0105] added roles `3`–`5`:

```
m / 1852' / 1815' / account' / 6 / generation'
```

The 64-byte extended private key at this path is the **key material**. The scalar used by a
purpose is derived from it by the algorithm profile's mapping, **domain-separated by purpose,
algorithm and network**:

```
tag = "cip????/" ‖ purpose ‖ "/" ‖ algorithm ‖ "/net" ‖ networkId ‖ "/v1"
sk  = Map_profile( key_material, tag )
PK  = sk · G
```

`networkId` is the decimal value returned by [CIP-30] `getNetworkId()` (`0` for test networks,
`1` for mainnet). Different purposes, profiles or networks therefore yield independent keys from
the same path, and a key used on a test network is never the mainnet key. All test networks share
`networkId` `0`, so Preview and Preprod share one key per purpose and profile.

**Keys of other CIPs on the same material.** Another CIP MAY derive keys of its own from the role-`6`
key material under a tag `cip????/<name>/…`, where `<name>` matches `[a-z0-9-]+` and is registered by
that CIP in the same identifier space as purposes, listed under [Purposes](#purposes). A name is never
both a purpose and such a registration, and neither contains `/`, so these tags never equal a
purpose's tag and no two registrations collide. Such keys are **not** keys of this
extension: the rules below for keys of this extension do not apply to them, and the CIP that
registers the name defines their use, their export rules and their consent. It MUST derive them
through HKDF with its own tag, so that they reveal nothing about any key of this extension.

- A key of this extension MUST be used only to compute ECDH shared secrets through this API.
  It MUST NOT be used to produce signatures or proofs whose form is `sk · P` for a point `P`
  derived from a message (for example BLS-style signatures), because `ecdh` would then act as a
  signing oracle.
- A key of this extension MUST NOT be, or be derivable from, a signing key or any signature the
  wallet produces on request.
- A wallet MUST derive `sk` per request, MUST erase it after use, and MUST NOT persist it.
- A wallet MUST NOT export the key material, or the extended private key of the role-`6` node,
  under any circumstances. The role level uses non-hardened derivation: with BIP32-Ed25519, a
  leaked child extended private key together with the account's public key reveals the account's
  private key.
- A wallet that cannot derive role `6` (hardware, multi-party, watch-only and similar wallets)
  MUST answer `Refused`. It MUST NOT substitute any other key.
- Accounts controlled by a script stake credential are out of scope.

### Algorithm Profiles

A wallet MUST reject an ephemeral key that fails its profile's validation rules **before** any
key material is derived or the user is prompted. Profile identifiers MUST match `[a-z0-9-]+`, so
that every `tag` decodes unambiguously.

| Profile | Point / secret encoding | Validation of `Epk` |
|---|---|---|
| `jubjub` | the `y` coordinate, 32 bytes big-endian; `S` returned as `S.y` | see [Appendix A](#appendix-a-jubjub-profile): canonical `y`, on the curve, in the prime-order subgroup, not the identity |

This CIP registers the `jubjub` profile only. Further profiles are added by a CIP that extends
this one.

### Purposes

A purpose is registered by this CIP, or by a later CIP that extends it. Each purpose registration
MUST define:

1. the purpose identifier (lowercase, `[a-z0-9-]+`);
2. the algorithm profiles allowed;
3. the format of `EcdhRequest.ref` and any scoping rules;
4. the effect-based wording of the consent prompt;
5. whether the key may also read outputs the account does not own, as a second reader of data
   sealed to it, and if so the consent wording that says it;
6. that its schemes are registered by the CIPs that define them (see [Schemes](#schemes)).

| Purpose | Status | Profiles | `ref` format |
|---|---|---|---|
| `viewing` | Specified below | `jubjub` | `"<txHash>#<outputIndex>"` of the output that carries `Epk` |

A name registered for keys outside this extension (see [Key Derivation](#key-derivation)) is
registered in the text of the CIP that uses it; a later CIP that registers a purpose or a name
checks that the name is not taken.

#### Purpose `viewing`

The key is the account's **viewing key**: a shared secret returned by `ecdh` opens one output
whose data was encrypted to the account.

- **`ref`:** MUST match `^[0-9a-f]{64}#(0|[1-9][0-9]*)$` and name the output that carries `Epk`.
- **Consent wording**, in substance:

  > «origin» is asking to read data that was encrypted to your account. It will not be able to
  > move funds. You can revoke this at any time.

  A wallet that confirms and serves second readers (see *Scope of disclosure* and *Confirming*)
  knows that it answers for outputs the account does not own. Its wording MUST say so by its
  effect: whose data the site can then read. A wallet that does not confirm MUST say the same
  whenever a registered scheme of the purpose has a second reader, because its grant lets a site
  read such outputs too: a wallet that does not scope answers for them directly, and one that
  scopes answers for an output planted at the user's address that carries the same `Epk` (`S` does
  not depend on the scheme). Only while no registered scheme of the purpose has a second reader may
  a wallet that does not confirm use the wording above unchanged.

- **Derivation from `S`, sender rules and recipient verification:** as the output's registered
  scheme defines them (scheme items 4–6). A dApp MUST apply the scheme's recipient check (scheme
  item 6), and MUST treat an output that fails it as unreadable.
- **Scope of disclosure:** the viewing key reveals the data of outputs **sent to** the account.
  A scheme MAY in addition seal an output's data to the viewing key of a second reader, an
  auditor for example. That key then also reveals the data of such outputs, which its account
  does not own.
  Whether data the account sent can be recovered depends on the scheme, not on this extension.
- **Scoping (MAY).** A wallet MAY answer only for outputs owned by the connected account or, if it confirms, sealed to it as a second reader. A wallet that scopes:
  - MUST resolve each `ref` from chain data it has verified itself (for example by checking that
    the producing transaction hashes to the referenced id), or from a transaction body the wallet
    itself signed, provided the referenced transaction id is that body's hash. Such signed bodies
    SHOULD expire after minutes, not days. Apart from such a body, which the wallet recorded itself
    when it signed it, it MUST NOT trust data supplied by the dApp.
  - MUST check that the output carries exactly the given `Epk` and is owned by the connected
    account, or, in a wallet that confirms and serves second readers, is of a format the scheme
    seals to a second reader.
  - MUST still answer for spent outputs, so that history remains readable.
  - MUST perform these checks **before** prompting the user, and report failures with the `info`
    codes in [Errors](#errors).
  - SHOULD rate-limit the chain lookups each origin can trigger, since they cost the wallet work
    before any prompt.

  Which outputs count as owned by the account, and where `Epk` sits in an output, are defined by
  the output's registered scheme (scheme items 2 and 3). An output that is not an output of a
  scheme the wallet supports counts as not owned (`not-own-ref`).

  Scoping alone narrows requests to keys that someone has placed at the user's address. **It does
  not prevent a chosen-key request**, because anyone can create an output shaped like the scheme's
  at the user's address carrying an `Epk` of their choosing. Confirming, below, closes that case.
  The [permission](#permission) remains the boundary for what an origin may read.
- **Confirming (MAY).** A scoping wallet that implements the scheme's opening rule (scheme
  item 7) MAY confirm each answer before releasing it. After computing `S`, and before it
  returns, stores or counts anything, it checks that `S` opens the output as it was resolved for
  scoping: the output's data decrypts under `S` and passes the scheme's check. A wallet that
  confirms:
  - MUST confirm every request of the call. If any output does not open, the whole call fails
    with `not-own-ref` naming those requests. The wallet then releases nothing.
  - MUST count an output whose format it cannot open as not owned, even when the format belongs
    to a scheme it supports, and so refuses it before prompting like any other output that is
    not owned.
  - MAY answer for an output the account does not own, when the scheme seals that output's data
    to the account's key as a second reader and `S` opens it in that role. Such an output MUST
    pass the checks above before prompting. The wallet MUST NOT answer for it without
    confirming, and only under a grant whose consent wording covers that case (see
    [Permission](#permission)).
  - MUST treat any error while confirming one request as that request not opening
    (`not-own-ref`), never as `InternalError`.
  - SHOULD make the work of the check independent of whether, and in which part, it fails.

  Confirming needs the key, so it is the one check that comes after the user interaction of a
  call. A wallet that confirms releases `S` only for an `Epk` whose output opens under `S`.
  Under the conditions of scheme item 7, making such an output needs `S` itself, or the
  opening of a genuine output on the same `Epk` that was sealed to the account in the same role.
  So the value released is one of three: a value the requester could compute; the value the
  wallet would release to the same origin for that genuine output's own reference; or a value
  the genuine output's sender, who knew it, made derivable by giving out its opening. No caller
  obtains `sk · P` for a point whose value nobody already holds.

### Schemes

A scheme is a concrete encryption format that uses a purpose's key. It is registered by a CIP that
names the purpose it serves, and that CIP defines the scheme's data; this CIP defines only what a
wallet needs from it. Each scheme registration MUST define:

1. the scheme identifier (lowercase, `[a-z0-9-]+`) and the purpose it serves;
2. which outputs belong to the scheme and where `Epk` sits in them;
3. what makes an output owned by an account, for scoping;
4. its roles (the owner and, optionally, a second reader) and, for each role, what it derives from
   `S`. Each derivation MUST bind `Epk`, and each role's derivation MUST be domain-separated from
   the others, so that an opening in one role gives none in another. With a fresh `e` (item 5), one
   shared secret then never yields the same keystream in two places;
5. the requirement on senders that `e` is fresh for every output and every transaction attempt,
   including rebuilt or fee-bumped attempts of the same payment;
6. a check the recipient MUST apply after decrypting, which rejects decrypted values not
   produced for this output (for example, that they agree with what the chain holds). A shared
   secret does not, by itself, make a ciphertext valid;
7. if wallets may confirm (see [Purpose `viewing`](#purpose-viewing)), the opening rule a wallet
   applies: the derivation from `S` of item 4 and the check of item 6. It MUST be such that an
   output on a given `Epk`, created without `S`, passes it with negligible probability, unless
   its author reuses the opening of a genuine output on that `Epk`.

A wallet that confirms answers only for outputs of schemes whose opening rule it implements, and
counts any other output as not owned. A wallet that only scopes may serve any scheme it supports
through items 2 and 3.

This CIP registers no scheme. Each scheme is registered in the CIP that defines it, which states
the purpose it serves and answers the items above.

### Permission

Use of a purpose is a permission **separate from** `enable()` and from every other purpose.

- The wallet MUST obtain the user's explicit consent before the first call for a purpose from an
  origin is answered. The prompt MUST use the purpose's effect-based wording and MUST NOT be
  presented as, or combined with, a signature prompt.
- A grant MUST be scoped at least to (wallet, origin, purpose, account index, generation).
- Grants MUST be listed where the user manages connected sites and MUST be revocable with
  immediate effect. Disconnecting an origin MUST revoke all its grants.
- Before releasing any result, the wallet MUST re-check that the grant still holds, still covers
  every request of the call, and that the active account is unchanged. If the active account
  changed, it MUST answer `AccountChange`.
- A grant covers what its consent wording said, and the wallet MUST record with each grant which
  wording it showed. Before prompting, the wallet determines from its scoping checks whether a
  call holds a request of a kind that wording did not cover, for example an output the account
  does not own. If so, the call's interaction MUST show the consent again with wording that
  covers it, and the grant is replaced only if the call is answered. Calls whose requests the
  recorded wording covers MAY continue under the earlier grant. This relies on senders using a
  fresh `e` (scheme item 5). A sender who reuses `e` for an output to the account and for
  an output sealed to it as second reader makes the answer for the first open the second.
- A wallet SHOULD require user interaction (for example the spending password) for each `ecdh`
  or `ecdhBatch` call. A wallet that answers without per-call interaction MUST rate-limit each
  origin, and SHOULD ask again when an origin's usage changes markedly.

Revocation is not retroactive: whatever an origin has already decrypted, it knows.

### Encryption Keys API

#### `api.cip????.getPublicKey(key: KeySelector): Promise<PublicKeyInfo>`

Errors: `APIError`

Returns the public key selected by `key`. A wallet MAY answer `Refused` for an `accountIndex`
other than that of the connected account.

#### `api.cip????.ecdh(request: EcdhRequest, key: KeySelector): Promise<HexString>`

Errors: `APIError`

Equivalent to `ecdhBatch([request], key)[0]`.

#### `api.cip????.ecdhBatch(requests: EcdhRequest[], key: KeySelector): Promise<HexString[]>`

Errors: `APIError`

For each request, validates `ref` under the purpose and `Epk` under the profile, and returns the
shared secret `S = sk · Epk`, encoded as defined by the profile.

- The result has one entry per request, **in input order**.
- The call is **all or nothing**: if any request fails validation, scoping or confirming, the
  whole call fails and no secret is returned. A failed call returns nothing, records no use of a
  grant, and creates, replaces or upgrades no grant.
- The wallet MAY cap the batch size. The cap MUST NOT be lower than 512 requests. A larger batch,
  or an empty one, is `InvalidRequest`.
- The wallet's obligations end at `S`. It need not interpret ciphertexts, keystreams or any
  other data of the scheme. A wallet that confirms (see [Purpose `viewing`](#purpose-viewing))
  reads them only to decide whether to release `S`.

### Errors

This extension reuses [CIP-30]'s
[`APIError`](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0030#apierror) codes.

| Condition | Code | `info` |
|---|---|---|
| Origin not connected, wallet locked, permission declined or revoked, wallet cannot derive role `6`, account, purpose or profile not available, or the extension is not offered on this network | `Refused` (−3) | free text |
| Malformed `KeySelector`; profile not allowed for the purpose; batch empty, not an array or over the wallet's cap; malformed `EcdhRequest` or `ref`; `Epk` fails the profile's validation rules | `InvalidRequest` (−1) | free text |
| One or more references are neither owned by the connected account nor, in a wallet that serves second readers, sealed to it as one; do not carry the given `Epk`; or, in a wallet that confirms, do not open under `S` (scoping wallets only) | `InvalidRequest` (−1) | `not-own-ref:[i,…]` |
| One or more references cannot be resolved yet (scoping wallets only) | `InvalidRequest` (−1) | `unresolved-ref:[i,…]` |
| The active account changed during the request | `AccountChange` (−4) | free text |
| Any other failure | `InternalError` (−2) | free text |

Indices are 0-based positions in that call's `requests` array, ascending, written as JSON
without spaces (e.g. `not-own-ref:[1,3]`). When both apply, only `not-own-ref` is reported.

dApps SHOULD validate every `Epk` and `ref` before calling, chunk large requests, and on
`not-own-ref` or `unresolved-ref` mark the affected requests and retry the rest as one batch.
dApps MUST NOT retry after `Refused`, and SHOULD NOT fall back to one call per request, since each
call may cost the user a prompt. A `not-own-ref` from a wallet that confirms arrives after the
user interaction, so the refused call may already have cost one. dApps SHOULD remember
references refused that way and not request them again.

### Versioning

This extension is versioned by its CIP number, as are all [CIP-30] extensions. A
backwards-incompatible change to the API requires a new extension CIP. Registering a new purpose
or algorithm profile is backwards-compatible: dApps select them by name, and wallets that do not
support them answer `Refused`. The `/v1` suffix in the derivation tag versions each key mapping
independently.

## Rationale: How does this CIP achieve its goals?

### Scope: three options considered

| Option | Scope | For | Against |
|---|---|---|---|
| A. One viewing purpose only | One key, one use | Smallest; already prototyped | Messaging or encrypted metadata would need another, overlapping CIP |
| **B. General ECDH, one key per purpose** (chosen) | One API, keys and permissions separated by purpose | Serves reading encrypted data now and other encryption uses later; a grant for one purpose never exposes another | Broader design; only one purpose has an implementer today |
| C. Wallet `encrypt` / `decrypt` | Wallet runs the whole scheme | Simplest for dApps | Ties wallets to each scheme's KDF and cipher; turns the wallet into a decryption oracle |

Option B keeps the security properties of A and leaves room for the uses that motivated a
general "asymmetric encryption" capability. It costs little: purposes are a naming and
derivation-tag layer over the same API. The choice is open to change on feedback.

### Why Jubjub only

The curve is dictated by whatever must check the encryption on chain. A scheme may prove, in a
Groth16 circuit over BLS12-381 verified with Plutus's BLS12-381 built-ins, that each output is
encrypted to its recipient's key. Jubjub is defined over BLS12-381's scalar
field, so its arithmetic is native to that circuit and costs a few thousand constraints per
operation. A curve over another field, such as ristretto255 over `2²⁵⁵ − 19`, would need
emulated field arithmetic in the circuit, typically one to two orders of magnitude more
constraints, and Plutus offers no built-ins for it. Such a curve only becomes useful if a future
ledger change verifies it natively. A CIP extending this one can then register it as a profile.

### ECDH, not decryption

The wallet computes the shared secret, not a plaintext. A `decrypt(ciphertext)` endpoint would
tie wallets to one scheme's key derivation, framing and cipher, and would have to change whenever
those do. ECDH between a long-lived key and a per-message ephemeral key is the primitive these
schemes share. It also keeps the unit of disclosure small: a shared secret opens **one** output
or message, whereas the key opens all of them. A wallet that confirms reads the scheme's data
only to decide whether to release the shared secret
(see [Confirming before release](#confirming-before-release)). It still returns no plaintext.

### A separate role, not the payment or stake key

Using the existing keys directly (Ed25519 converted to X25519) means one key both signs and
encrypts. That is the kind of key reuse MetaMask deprecated, and it is proven jointly secure only
under specific conditions ([ePrint 2021/509]). It also cannot serve `jubjub` at all. Deriving the
encryption key *from* the stake key would be cryptographically sound, but this CIP forbids it (see
[Key Derivation](#key-derivation)) because it loses what a separate branch keeps:

1. **Rotation.** The stake key has no generation index. Rotating the encryption key would mean
   rotating the stake key, which changes addresses and delegation.
2. **Damage limited by structure.** The key material sits at a hardened node on its own branch.
   If it leaks through an implementation bug, it reveals no stake or spending key. A key derived
   from the stake key has that key as its input, and a leaked stake key together with the
   account's public key reveals the account's private key.
3. **One branch for the ecosystem.** [CIP-0105] added three roles for governance keys. One more
   role covers every encryption use, rather than one ad hoc derivation per application. Other
   CIPs that need keys tied to the same account (a shielded pool's spend and proving keys, for example)
   register a name under the same branch instead of claiming another role (see
   [Key Derivation](#key-derivation)).

Discovery is not an argument for the existing keys either. A stake public key is visible on chain
only after its owner has signed with it, and a `jubjub` key cannot be read off it at all. A claim
signed with [CIP-30] `signData` does that job (see [No export](#no-export-proof-of-possession-is-open)).

### Separate keys per purpose and per network

Reusing one key across purposes would let a site granted access to, for example, encrypted
messages read the data encrypted to the user. Domain separation in the key mapping gives each
purpose an independent key from the same seed and path, so no new backup is needed. Separate
permissions per purpose make the user's consent meaningful.

The network is part of the tag for a similar reason. Without it, a test-network dApp could
plant, at the user's test-network address, an output copied from a mainnet output sealed to that
user, and have it opened after consent. With per-network keys, a mainnet output cannot be opened
by the test-network key.

### Static-key ECDH oracle

Without confirming, the API is an oracle that returns `sk · P` for points the caller chooses
(scoping only requires that someone placed the point at the user's address). A wallet that
confirms is not (see [Confirming before release](#confirming-before-release)). Two consequences
shape the specification for the others:

- **Key-usage restriction.** `sk · H(m)` is a BLS-style signature on `m`, so the key MUST NOT be
  used for any signature or proof of that form. Schnorr-style proofs, whose responses are scalars,
  are not affected.
- **Repeated queries.** A static Diffie–Hellman oracle allows the Brown–Gallant / Cheon attack,
  which lowers the cost of recovering `sk` when the group order `ℓ` satisfies `d | ℓ − 1` for a
  usable `d` and the attacker can chain about `d` queries. For Jubjub, the small factors of
  `ℓ − 1` are `2 · 3 · 12281` (`d ≈ 2¹⁶`), so about 74 000 chained queries would cut security
  by roughly 8 bits, from about 126 bits. This is negligible, but it is why wallets that answer
  without per-call interaction MUST rate-limit.

### No export; proof of possession is open

- **No key export.** It would reduce the design to "any site that obtained it once has it
  forever". If an auditor ever needs a secret itself, that is a separate and much more severe
  permission, and wallets must remain free not to offer it.
- **Binding a key to an account.** [CIP-30] `signData` already lets a stake key sign a claim
  containing `PK`, since it specifies that *"the staking key will be used for reward
  addresses"*. That proves authority over the **account**. It does **not** prove that anyone holds
  the private key behind `PK`. A buggy wallet, or an account owner setting a trap, could bind a
  key that nobody can use, and payments sent to it would be stranded. Whether this extension
  should also offer a proof of possession is [Open Question 2](#open-questions).

### References, not bare keys

Requiring a `ref` with every `Epk` costs the dApp nothing, because it read `Epk` from that place.
It lets wallets scope requests, reject garbage before prompting, and keep an audit trail. A
bare-key form would make the endpoint a general-purpose ECDH oracle on the key.

The scheme's derivation binds `Epk`, not `ref` (scheme item 4). A sender cannot bind the
transaction hash in `ref`: the transaction body covers every output, its datum or datum hash and
the auxiliary-data hash, so the hash is known only after the ciphertext is written. The output
index could be bound, but it adds nothing once `e` is fresh for every output and attempt (scheme
item 5): a fresh `e` already gives every output its own `Epk` and so its own keystream.

### Confirming before release

Scoping by ownership checks where an `Epk` sits, never who chose it. Anyone can pay an output
shaped like the scheme's to the user's address with an `Epk` of their choosing, and a wallet that
scopes by shape then answers `sk · Epk` for it. Confirming removes that case: the wallet releases
`S` only where the output opens under `S`, which needs `S` or a genuine opening on the same `Epk`
(see [Confirming](#purpose-viewing)). The static Diffie–Hellman attacks need answers on points
whose answer nobody holds, and confirming gives none.

It has two costs, which is why it is optional. The wallet must implement the opening rule of each
scheme it serves, so a new output format needs wallet support before a dApp can ask for it. This
is the coupling that [ECDH, not decryption](#ecdh-not-decryption) avoids for the result, accepted
here for the release decision only. And because the check needs the key, an output planted at
the user's address can cost the user one interaction for a call that is then refused.

Confirming is also what makes a second reader safe to serve. A wallet that answered for outputs
its account does not own without it would hand `sk · Epk` to anyone who can name an output. A
wallet that knows no scheme's opening rule can still scope by ownership; the permission bounds
what an origin reads either way.

### Point validation

The `Epk` is the only place where a page can attack the **key** rather than the data. Jubjub has
cofactor 8, so it has small-order points, and a `y`-only encoding admits values that are not on
the curve at all. A crafted `Epk` of either kind can leak `sk` modulo a small factor: through a
small subgroup, or through an "invalid-curve" computation on the curve's quadratic twist. The
checks in [Appendix A](#appendix-a-jubjub-profile) close both, before any key material is
touched. This is acceptable for a cofactor curve here because the key is used for one operation
only, on validated points, with no homomorphic algebra on attacker-supplied values. It also means
a hostile page cannot cost the user a prompt with an invalid key.

### Privacy

A purpose's public key is a stable identifier for the account across every origin and application
that uses that purpose, and wherever it is published with a stake-key claim it is linked to the
account. Per-network keys add no link across networks. A new generation gives a key that origins
without a grant for the old one cannot link to it. It is still linked if bound by the same
stake-key claim, and it rotates every purpose (see [Open Question 1](#open-questions)).

### Relationship to other proposals

- **[CIP-95]** is the template: a namespaced, additive [CIP-30] extension that introduced a new
  key role together with wallet behaviour for it.
- **[CIP-1852] / [CIP-0105]:** role `6` follows the way [CIP-0105] added governance roles. The
  derivation could be split into a separate companion CIP if wallet implementors prefer.
- **[CIP-83]** is a candidate for a future purpose that would replace shared passphrases with
  public-key encryption.
- **Out of scope: authorising spends.** Whether and when a wallet witnesses a transaction that
  spends outputs encrypted to it is a question of the scheme's or the asset's signing rules, not of
  key agreement.

### Backwards Compatibility

This proposal is purely additive. No existing [CIP-30] endpoint changes. dApps and wallets that
do not use the extension are unaffected.

### Open Questions

1. **Path level or tag for purposes.** Purposes are separated in the key mapping, not in the path.
   This keeps one generation index per account, so rotating one purpose's key rotates all of them.
   Should purposes instead get their own path level (`…/6/purpose'/generation'`)?
   *Authors' current answer:* keep the tag. One generation index is simpler for wallets, and the
   one registered purpose has no reason to rotate on its own; a purpose that needs independent
   rotation can be given its own level by the CIP that registers it.
2. **Proof of possession.** Should the extension offer an endpoint that proves the wallet holds
   `sk` for a published `PK`, such as a Schnorr proof with the account's stake credential bound
   into the challenge? Or is that left to each scheme? A scheme whose on-chain check proves only
   that an output decrypts under the key its owner signed does not detect a key nobody holds.
   *Authors' current answer:* leave it to each scheme. A scheme that needs it can prove
   possession in its own way, for example inside its proof circuit, and a generic endpoint would
   add a signing use of the key that this CIP otherwise forbids.
3. **Generation discovery.** dApps have no way to learn which generations an account has used.
   Should `getPublicKey` report the latest generation, or should purposes define how generations
   are discovered (for example from on-chain registrations)?
   *Authors' current answer:* the purpose or its schemes define it; this CIP adds no discovery
   call.
4. **Sender-side derivation.** The `jubjub` profile lets the dApp choose the ephemeral scalar `e`.
   A future scheme may require the sender to derive `e` from its own key, for example to make
   outgoing payments auditable, and a dApp cannot do that without the wallet. Should purposes be
   able to define a sender-side endpoint? *Authors' current answer:* yes, by a later CIP that
   registers such a purpose; it does not block `viewing`.
5. **Hardware wallets.** Role-`6` derivation is ordinary BIP32-Ed25519 work, but current Cardano
   hardware-wallet apps expose no on-device ECDH. Should the CIP define how a hardware wallet
   signals this, beyond `Refused`? *Authors' current answer:* not yet; `Refused` is enough until
   a device offers ECDH, and that device's CIP can add a signal.
6. **A second purpose.** Should an encrypted-message purpose for [CIP-83]-style metadata be
   specified in this CIP, or left to a follow-up? *Authors' current answer:* a follow-up, so that
   this CIP stays reviewable; the purpose and scheme registries are built to take it.

Two earlier questions are closed in this version: whether a scoping wallet must confirm (no; it
reports `capabilities.confirming`, and only confirming wallets may serve second readers), and how
a wallet that does not confirm words its consent when a scheme has a second reader (it says so by
its effect, see [Purpose `viewing`](#purpose-viewing)).

## Path to Active

### Acceptance Criteria

- [ ] A CIP number is assigned, and role `6` (or, if role `6` is taken, the next free role, with
      no other change) is accepted into [CIP-1852], either by this CIP or by a companion CIP.
- [ ] Open Questions 1 and 2 are resolved in the specification.
- [ ] Conformance test vectors are published for every registered purpose and profile: key
      derivation for each network, `PK`, ECDH, and rejection of invalid `Epk` values.
- [ ] An independent cryptographic review of the specification is completed.
- [ ] At least two wallets implement the extension under its assigned namespace.
- [ ] At least one dApp uses the extension in production.

### Implementation Plan

- [x] Prototype of the `viewing` purpose with the `jubjub` profile in Gero Wallet (an
      experimental branch; Preview and Preprod only, mainnet refused).
- [x] Reference dApp using the prototype to open outputs encrypted to the account, verified on
      Preprod.
- [x] Align the prototype with this specification (namespace under the provisional number
      `9001`, method names, derivation tag with network separation, explicit on-curve check).
- [ ] Report `capabilities` from the prototype's `getPublicKey`.
- [x] Reference implementations in TypeScript (`@noble/curves`) and Java (cardano-client-lib and
      BouncyCastle) that produce the conformance test vectors
      ([Appendix B](#appendix-b-test-vectors)).
- [ ] Engage other wallet implementors via the wallet working groups and [CPS-0010].

## References

- [CIP-8] — Message Signing
- [CIP-30] — Cardano dApp-Wallet Web Bridge
- [CIP-83] — Encrypted Transaction message/comment metadata
- [CIP-95] — Web-Wallet Bridge - Conway ledger era
- [CIP-0105] — Conway era Key Chains for HD Wallets
- [CIP-113] — Programmable token-like assets
- [CIP-1852] — HD (Hierarchy for Deterministic) Wallets for Cardano
- [CPS-0010] — Wallet Connectors
- [ePrint 2021/509] — On using the same key pair for Ed25519 and an X25519 based KEM
- [ERC-5630] — New approach for encryption / decryption
- [Issue #204] — `api.getSharedSecret` for CIP-30
- [RFC 2119] — Key words for use in RFCs to Indicate Requirement Levels
- [RFC 5869] — HMAC-based Extract-and-Expand Key Derivation Function (HKDF)

## Appendices

### Appendix A: Jubjub profile

- **Group:** the prime-order subgroup of Jubjub (order `ℓ`), the twisted Edwards curve
  `−x² + y² = 1 + d·x²·y²` defined over the BLS12-381 scalar field `p`, with cofactor 8. Used where
  a circuit over that field must prove that data is encrypted to a given key.
- **Encoding:** points are represented by their `y` coordinate, 32 bytes big-endian. The sign of
  `x` is irrelevant to the result, since `sk·(±Epk)` has the same `y`.
- **Mapping:** with HKDF-SHA256 as in [RFC 5869],

  ```
  sk = HKDF-SHA256( ikm  = key_material (64 bytes),
                    salt = none,
                    info = tag,
                    L    = 64 ) mod ℓ
  ```

  The key material is the 64-byte extended private key `kL‖kR` of the role-`6` node, without its
  chain code. The 64 output bytes are read as a big-endian integer and reduced mod `ℓ`; `sk` is
  then encoded as 32 bytes big-endian.

  If the result is `0`, which happens with negligible probability, the wallet MUST answer
  `InternalError`.
- **Validation of `Epk`.** The wallet MUST reject `Epk` unless all of the following hold, in this
  order:
  1. it is exactly 32 bytes, and `y < p`;
  2. **on the curve:** `x² = (1 − y²) / (−1 − d·y²)` has a solution in the field, i.e. the
     right-hand side is defined and is a square. A value of `y` that fails this check has no point
     on Jubjub, and computing with it on a `y`-only ladder would operate on the twist;
  3. **in the prime-order subgroup:** for either recovered point `P`, `ℓ · P` is the identity.
     This excludes every small-order point other than the identity (`y = p − 1`, `y = 0` and the
     two `y` values of the order-8 points) and every mixed-order point;
  4. **not the identity:** `y ≠ 1`.

  The points of order 1, 2, 4 and 8 have five `y` values. A wallet MAY reject them by value before
  step 2, as a cheap first filter. Step 3 or 4 rejects them in any case (big-endian hex):

  ```
  0000000000000000000000000000000000000000000000000000000000000001   order 1 (identity)
  73eda753299d7d483339d80809a1d80553bda402fffe5bfeffffffff00000000   order 2 (y = p − 1)
  0000000000000000000000000000000000000000000000000000000000000000   order 4
  4958bdb21966982e16a13035ad4d72669106ee90f384a4a1ff0d2068eff496dd   order 8
  2a94e9a11036e51a1c98a7d25c54659ec2b6b5720c79b75d00f2df96100b6924   order 8
  ```

### Appendix B: Test vectors

[`test-vectors.json`](./test-vectors.json) holds conformance vectors for purpose
`viewing`, profile `jubjub`: five vectors over two seed phrases, giving the key
material, HKDF output, `sk`, `PK.y` and one ECDH result each, and covering networkId 0 and 1,
account 1 and generation 1, plus nine encodings of `Epk` that a wallet MUST reject. The tag in every
vector holds the placeholder `cip????`, so every value changes once this CIP is numbered. They are
reproduced independently by a TypeScript implementation (`@noble/curves`), a Java one
(BouncyCastle) and Gero Wallet's prototype. Confirming depends on each scheme's opening rule,
so its vectors belong to the scheme, not to this CIP.

## Acknowledgements

The API design and the `jubjub` prototype come from a proof of concept and its Gero Wallet
integration. The security requirements draw on review findings by Dr. Manvir Schneider.

## Copyright

This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).

[CIP-8]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0008
[CIP-30]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0030
[CIP-113]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0113
[CIP-83]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0083
[CIP-95]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0095
[CIP-0105]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-0105
[CIP-1852]: https://github.com/cardano-foundation/CIPs/tree/master/CIP-1852
[CPS-0010]: https://github.com/cardano-foundation/CIPs/tree/master/CPS-0010
[ePrint 2021/509]: https://eprint.iacr.org/2021/509
[ERC-5630]: https://eips.ethereum.org/EIPS/eip-5630
[Issue #204]: https://github.com/cardano-foundation/CIPs/issues/204
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 5869]: https://www.rfc-editor.org/rfc/rfc5869
