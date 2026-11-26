## Preamble

```
SEP: To Be Assigned
Title: Contract Display Descriptors
Author: Jun Luo <@overcat>, <add your name here>
Status: Draft
Created: 2026-11-02
Updated: 2026-11-02
Version: 0.0.1
Discussion: TBD
```

> **Disclosure:** an AI assistant was used to polish the wording of this draft.

> **TODO (pre-submission draft):** open the GitHub Discussion for this proposal
> and fill in the `Discussion` field above.

## Simple Summary

A small, off-chain JSON format that tells a wallet or explorer how to present a
Soroban contract function call to a human: a short intent phrase, labels for
the parameters that matter, and formatting rules for amounts and times.
Descriptors are bound to a contract's Wasm hash and validated against the
contract's [SEP-48] interface specification before use. Stellar Asset Contract
`transfer`, `approve`, `burn` and `trust` calls, which have no Wasm, are
rendered by a fixed built-in rule.

This first version is kept small on purpose. It covers calls whose meaning can
be read from values at fixed positions in the arguments, at the top level or
inside vectors, tuples and structs. Examples are [SEP-41] token transfers and
approvals, including those of Stellar Asset Contracts, and swaps and deposits
whose tokens appear in the arguments. Every other call falls back to the raw
display, which is mandatory.

[SEP-48]:
  https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0048.md
[SEP-41]:
  https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0041.md

## Dependencies

- [SEP-48]: Contract Interface Specification. Descriptors reference function
  and parameter names from the `contractspecv0` section, and wallets validate
  descriptors against it.
- [SEP-41]: Token Interface. The `amount` format resolves `decimals` and
  `symbol` via the SEP-41 interface; §5.3 renders Stellar Asset Contract calls
  using SEP-41 semantics.
- [SEP-23]: Strkeys. Address rendering.
- [CAP-71]: Soroban authorization credentials `SOROBAN_CREDENTIALS_ADDRESS_V2`
  and `SOROBAN_CREDENTIALS_ADDRESS_WITH_DELEGATES`, and the signature payload
  that binds the authorizing address.
- [CAP-73]: The Stellar Asset Contract `trust` function, which §5.3 renders
  with a built-in rule.
- [CAP-84]: Muxed contract addresses, accepted by the argument check in §4.4.
- [CAP-85]: External contract executables, resolved in §4.1.
- [CAP-86]: Sparse struct maps, which let a struct value omit an empty member
  (§4.4).

[SEP-23]:
  https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0023.md
[CAP-71]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0071.md
[CAP-73]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0073.md
[CAP-84]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0084.md
[CAP-85]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0085.md
[CAP-86]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0086.md

## Motivation

Wallets can already decode the structure of a Soroban invocation. The [SEP-48]
spec in the Wasm gives every argument a name and a type. Varial reads it and
uses it to label arguments, and the Ledger and Trezor Stellar apps parse
`SCVal` values and authorization trees on the device.

That is not enough to tell a user what a call does.
`transfer(from, to, amount = 1000000000)` does not show that the user is
sending 100 USDC. The spec does not say which token defines the units of
`amount`, or that `live_until_ledger` is an expiry. Its `doc` field is a
comment for programmers with no machine-readable semantics, and no wallet we
know of displays it.

Other ecosystems have tried three ways to add that meaning. The first was to
let the contract describe itself in free text (Solidity NatSpec, EIP-3224,
ERC-4430). It was abandoned because a malicious contract can write a reassuring
description too. The second was to make readable text part of the signed
payload (Cosmos SDK `SIGN_MODE_TEXTUAL`). It never reached adoption and was
removed from the SDK in 2026. The third is ERC-7730, which shipped and is now
maintained by the Ethereum Foundation and used by Ledger and Trezor. It is a
small, curated, off-chain descriptor bound to the contract's code, with a fixed
set of formatters and an explicit fallback when no descriptor exists.

Stellar has no such format yet, and this SEP proposes one. It can be simpler
than ERC-7730 because of what Soroban already provides. On the EVM a descriptor
has to supply the ABI itself and bind to a mutable contract address, because
the chain stores neither the interface nor a stable identity for the code. On
Soroban the [SEP-48] spec is part of the Wasm, so a descriptor carries no ABI,
is validated against the real interface, and binds to the immutable Wasm hash.

## Abstract

This SEP defines:

1. A JSON **descriptor document** bound to one or more Wasm hashes, containing
   a per-function **intent** phrase, an **authorization assertion**, and an
   ordered list of **fields** that label and format parameters.
1. Six **formats**, `amount`, `timestamp`, `duration`, `percent`, `bool` and
   `enum`, covering the parameters whose raw values are misleading to humans.
1. **Validation rules** that a wallet applies before rendering a descriptor,
   tying it to the SEP-48 spec of the Wasm it claims to describe, and a single
   **argument check** shared by host function invocations and authorization
   nodes.
1. **Rendering requirements** for the host function invocation, for
   `SorobanAuthorizedInvocation` nodes in the authorization tree, and a
   built-in rule for Stellar Asset Contract `transfer`, `approve`, `burn` and
   `trust`.
1. A mandatory **fallback** when no valid descriptor applies.

This version does not define a registry, a signing or attestation scheme, a
hardware-wallet transport encoding, or any simulation-based effect preview. See
[Limitations](#limitations).

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted
as described in RFC 2119.

This document says "wallet" for brevity, but its requirements apply to any
consumer that presents a Soroban invocation to a human: signing wallets,
hardware signers, block explorers, transaction builders. A consumer rendering
an already-executed transaction MUST perform the resolution in §4.1 against the
ledger state as of that transaction's ledger, not the current state, so that a
contract upgraded since then is rendered with the descriptor for the code that
actually ran.

In this document the **host** of a request is the application that passes it to
the wallet: a dApp calling a wallet, or the companion application of a hardware
signer. The term is unrelated to the protocol's _host function_.

Spec type names used in this document are the `SCSpecType` enum names without
the `SC_SPEC_TYPE_` prefix, in snake_case (the SEP-51 XDR-JSON form): `u32`,
`i128`, `timepoint`, `address`, `muxed_address`, `vec`, and so on.

All `intent` and `label` strings MUST be non-empty and consist of printable
ASCII (U+0020 to U+007E). No maximum length is imposed; curators SHOULD keep
them short enough for a hardware-wallet screen, and wallets MAY truncate an
overlong string with an ellipsis provided the full text remains viewable.

### 1. Descriptor document

A descriptor document is a UTF-8 encoded JSON object.

| field         | type     | required | description                                                                                                                                                                  |
| ------------- | -------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version`     | integer  | yes      | Format version. This SEP defines version `1`. Wallets MUST ignore documents with an unknown version.                                                                         |
| `wasm_hashes` | string[] | yes      | One or more SHA-256 hashes of the contract Wasm bytes (the `ContractCodeEntry` hash), each exactly 64 lowercase hex characters. The descriptor applies to every listed Wasm. |
| `functions`   | object   | yes      | Map from function name (as it appears in the SEP-48 spec) to a [function descriptor](#2-function-descriptor). Functions not present in the map have no descriptor.           |

Unknown top-level keys MUST be ignored, to allow forward-compatible extensions.
Unknown keys inside a function or field descriptor invalidate that function
descriptor (§4.3). Extensions that change rendering or narrow where a
descriptor applies MUST bump `version`.

A document in which any JSON object has duplicate keys is invalid as a whole: a
wallet MUST use a parser that rejects duplicate keys or otherwise treat the
document as invalid.

A descriptor document has no network field. Wasm hashes are network
independent; contract IDs are not, which is why a descriptor never names one.

### 2. Function descriptor

| field    | type   | required | description                                                                                                                                                                                                                                                                                       |
| -------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `intent` | string | yes      | A short English phrase naming the action from the signer's perspective, e.g. `Send tokens`, `Approve token spending`, `Swap tokens`, `Deposit into lending pool`. It contains no values. It SHOULD NOT name a protocol or organization, since a descriptor applies to every instance of the Wasm. |
| `auth`   | array  | no       | Path (§2.1) to the argument that holds the authorizing address. Asserts that the function demands authorization only from that address, over exactly its own arguments; defined below. Governs §5.2.                                                                                              |
| `fields` | array  | no       | Ordered list of [field descriptors](#21-field-descriptor). Order defines display order. Parameters not listed are not part of the described view; they remain available in the raw view (§5.1).                                                                                                   |

`auth` exists because an authorization node is not a record of a call. A
`SorobanAuthorizedInvocation` carries a contract, a function name and an
argument list, but that list is whatever the contract passed when it demanded
authorization:

- `from.require_auth()` records the function's own arguments.
- `from.require_auth_for_args(...)` records an arbitrary list under the same
  function name.

The spec describes only the function's declared parameters and does not say
which of the two a function does, so a wallet that labelled every authorization
node with the spec's parameter names would mislabel the second kind. Varial hit
exactly this
([stellar/freighter#2196](https://github.com/stellar/freighter/issues/2196): a
function declared `(from: Address, id: u64)` authorized
`(token: Address, id: u64)`, and the token contract was labelled "From") and
now labels no authorization node from the spec.

`auth` is the curator's statement, after reading the source, that the function
demands authorization in exactly one way: it calls `require_auth()` on the
address at the `auth` path, and it demands authorization from no other address
and over no other argument list. Every authorization node for this function,
whether it appears as a root or as a sub-invocation, then carries exactly the
function's arguments in order and is authorized by the address at that path. A
wallet renders such a node with this function descriptor only when that address
is the entry's authorizer (§5.2).

- `auth` says nothing about the authorizations that the function's own
  sub-invocations require; those are governed by the descriptors of the
  contracts they call.
- A function whose authorizer is not one of its arguments (an administrator
  read from storage, for example), or that demands authorization from more than
  one address, cannot set `auth`. Its authorization nodes are rendered raw.
- Curators MUST verify the assertion against the contract source before setting
  it. The wallet's checks (§4.4, §5.2) catch a wrong assertion only when the
  argument count, the type of a listed field, or the address at the `auth` path
  differs.

> **TODO (pre-submission draft):** `intent` is a fixed phrase; values appear
> only as labelled lines below it. ERC-7730 additionally allows an optional
> templated sentence (`Send {value} to {to}`) that falls back to the fixed
> phrase when a value cannot be filled in. Whether to add such a template, and
> how it should behave on small screens (unresolved tokens, abbreviated
> addresses, compound values), is undecided.

> **TODO (pre-submission draft):** reviewers suggested standard intent
> identifiers (for example `token.transfer`) that a wallet maps to text in the
> user's language, instead of an English phrase written by the curator. This is
> still to be done. It needs a shared list of identifiers and a rule for
> intents that the list does not cover.

#### 2.1 Field descriptor

| field           | type    | required | description                                                                                                                                    |
| --------------- | ------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `path`          | array   | yes      | Path to the value the field shows (see below). Two fields MUST NOT have the same path.                                                         |
| `label`         | string  | yes      | Human-readable label, e.g. `Amount`, `Recipient`, `Expires`.                                                                                   |
| `format`        | string  | no       | One of `amount`, `timestamp`, `duration`, `percent`, `bool`, `enum` (§3). When absent, the value is rendered raw according to its SEP-48 type. |
| `token`         | object  | cond.    | Present if and only if `format` is `amount`. See §3.1.                                                                                         |
| `decimals`      | integer | cond.    | Present if and only if `format` is `percent`. See §3.4.                                                                                        |
| `true`, `false` | string  | cond.    | Both present if and only if `format` is `bool`. See §3.5.                                                                                      |
| `values`        | object  | cond.    | Present if and only if `format` is `enum`. See §3.6.                                                                                           |

A **path** is a JSON array that locates one value in the arguments:

- The first element is the name of a function input.
- An integer steps into a `vec`, a `tuple` or a tuple struct. Negative values
  count from the end, so `-1` is the last element.
- A string steps into a member of a struct that the spec declares as
  `SC_SPEC_ENTRY_UDT_STRUCT_V0`.

| path                     | names                                                      |
| ------------------------ | ---------------------------------------------------------- |
| `["amount"]`             | a top-level input                                          |
| `["swaps_chain", -1, 2]` | the third member of the last element of a vector of tuples |
| `["params", "asset"]`    | the `asset` member of a struct input                       |

A **tuple struct** is a struct whose `n` members are named exactly `0` to
`n-1`, in that order (soroban-sdk declares a Rust tuple struct this way and
encodes its value as a vector). An integer steps into it as into a `tuple`; a
string does not.

A path cannot continue into a value whose spec type is `map` or `option`. The
keys of a map are data that the spec does not declare, and an option may be
empty. Except for a tuple struct, a struct value is also encoded as an
`SCV_MAP`, but the spec declares its members, so a path can step into them. A
path can end at a `map` or `option` value, but no format accepts these types,
so the field shows it raw (§3.0).

There is no construct that describes every element of a variable-length vector
at once: a curator who wants per-element display lists each position
explicitly, and §4.4 treats an index beyond the vector's actual length as an
omitted field rather than an error.

### 3. Rendering of values

A format changes how the described view shows a value. The value itself MUST
remain viewable in the raw view (§5.1).

> **TODO (pre-submission draft):** this version defines six formats, `amount`,
> `timestamp`, `duration`, `percent`, `bool` and `enum`, chosen from the
> token-transfer, swap, lending and locking call shapes surveyed so far. Before
> formal submission, survey deployed Soroban contracts for other parameter
> kinds whose raw rendering misleads users (byte strings, …) and decide which,
> if any, earn a format in v1.

#### 3.0 Raw rendering

Parameters without a format, all parameters of undescribed calls, and the raw
view of a described call (§5.1) are rendered raw:

- Addresses render as their [SEP-23] strkey. A wallet MAY abbreviate a strkey
  on screen, and MAY add a label from the user's address book or another
  trusted name source, only if the full strkey, including any muxed ID, remains
  viewable.
- Integers render as decimal integers.
- Everything else renders in the wallet's existing representation.

#### 3.1 `amount`

Accepted spec types: `i128`, `u128`, `i64`, `u64`, `i32`, `u32`.

The parameter is a token quantity. The `token` object identifies which token
contract defines its units. Exactly one of the following shapes MUST be used:

| shape               | meaning                                                                                            |
| ------------------- | -------------------------------------------------------------------------------------------------- |
| `{ "path": [...] }` | A path (§2.1) whose end is of spec type `address`; its value is the token contract ID.             |
| `{ "self": true }`  | The called contract is itself the token (e.g. a SEP-41 token's own `transfer`, `approve`, `burn`). |

To render an `amount`, a wallet resolves the token's `decimals` and `symbol`
from a source it trusts, for example a curated token list bundled with the
wallet (the only practical option for an offline signer) or the token's own
SEP-41 `decimals()` and `symbol()` functions. Values read from the contract are
unverified metadata: a contract can return anything, and different values at
different times, so wallets SHOULD prefer a curated list and SHOULD mark
contract-sourced metadata as unverified. For a token that is a Stellar Asset
Contract, §5.3 gives the `decimals` and `symbol` instead.

The rendered value is exactly `value / 10^decimals` in base 10: no rounding, no
abbreviation, no exponent notation, a leading `-` for negative values. The
decimal separator is `.`. Trailing fractional zeros MAY be omitted. Digit
grouping MAY be added, using `,` (or a thin space) every three integer digits.
The `symbol` follows the number. Decimals above an implementation limit (38 is
reasonable) are treated as unresolved.

If the wallet cannot resolve `decimals`, it MUST render the parameter as the
raw integer, MUST mark it as unscaled (e.g. `1000000000 (raw units)`), and MUST
NOT guess a scale. If it cannot resolve `symbol`, it MUST render the token's
contract ID in place of a symbol.

If a wallet obtains `decimals` for a token from more than one source and the
values disagree, it MUST NOT render a scaled amount from any of them. This is a
check failure (§4.4).

A `symbol` from any source MUST be treated as untrusted text. SEP-41 `symbol()`
returns a `String`, an unbounded XDR `SCString` of arbitrary bytes that the
protocol does not require to be valid UTF-8, let alone printable; it is not the
constrained `Symbol` type. The following rules apply:

- It MUST be non-empty and consist of printable ASCII (U+0020 to U+007E), the
  same rule as for `intent` and `label`. A `symbol` that breaks this rule MUST
  NOT be displayed. This is a check failure (§4.4).
- It SHOULD be truncated to a bounded length (32 bytes is reasonable) with an
  ellipsis.
- It MUST NOT be able to introduce new lines or fields in the rendered layout.

> **TODO (pre-submission draft):** the ASCII rule rejects any symbol written in
> another script, and the protocol allows such symbols because `symbol()`
> returns arbitrary bytes. Survey deployed tokens, including NFT collections,
> for non-ASCII symbols and decide whether the rule should admit more than
> ASCII.

#### 3.2 `timestamp`

Accepted spec types: `u64`, `timepoint`.

The parameter is seconds since the Unix epoch. The wallet renders it as an RFC
3339 date and time in UTC (e.g. `2026-11-23T14:00:00Z`); it MAY add local time.
A value greater than `253402300799` (`9999-12-31T23:59:59Z`) has no RFC 3339
form and is a check failure (§4.4).

#### 3.3 `duration`

Accepted spec types: `u32`, `u64`, `duration`.

The parameter is a length of time in seconds. The wallet renders it as days,
hours, minutes and seconds, omitting every unit that is zero and never using
months or years, whose lengths vary: `2592000` renders as `30d`, `5400` as
`1h 30m`, `3601` as `1h 1s`, `90061` as `1d 1h 1m 1s`. A zero duration renders
as `0s`.

#### 3.4 `percent`

Accepted spec types: `i128`, `u128`, `i64`, `u64`, `i32`, `u32`.

The parameter is a ratio stored as an integer scaled by a power of ten. The
field's `decimals` (an integer from 0 to 38) gives the scale: the ratio is
`value / 10^decimals`. The wallet renders the ratio multiplied by 100, exactly
and with no rounding, followed by `%`; trailing fractional zeros MAY be omitted
and a leading `-` marks negative values. With `decimals: 4` (a denominator of
10,000, one basis point per unit) the value `30` renders as `0.3%`; with
`decimals: 7` the value `1250000` renders as `12.5%`.

The unit is fixed to `%`. A general unit format with a curator-supplied symbol
was considered and rejected because the symbol could be set to a token code,
bypassing the token resolution and verification rules of §3.1 and §5.3.

#### 3.5 `bool`

Accepted spec type: `bool`.

The field's `true` and `false` strings (subject to the charset rule for labels)
are displayed in place of the boolean value, e.g. `Enabled` / `Disabled` or
`Yes` / `No`.

#### 3.6 `enum`

Accepted spec types: `u32`, `i32`, `u64`, `i64`.

The field's `values` object maps the decimal string form of an integer to a
phrase (subject to the charset rule for labels), e.g.
`{ "2": "Supply collateral", "4": "Borrow" }`. The matching phrase is
displayed. A value with no entry is a check failure (§4.4), because a code the
curator did not list often changes what the neighbouring arguments mean.

### 4. Validation

#### 4.1 Resolving a contract to a Wasm hash

A contract instance's `executable` is resolved as follows:

- `CONTRACT_EXECUTABLE_WASM`: the Wasm hash is `wasm_hash`.
- `CONTRACT_EXECUTABLE_EXTERNAL_REF` ([CAP-85]): the Wasm hash is the value of
  the persistent contract data entry of `executable_owner` keyed by
  `SCV_EXECUTABLE_TAG(tag)`; the entry's value MUST be `SCV_BYTES` of
  length 32.
- `CONTRACT_EXECUTABLE_STELLAR_ASSET`: no Wasm hash. §5.3 applies.

If resolution fails, no descriptor applies.

#### 4.2 Selecting a descriptor

A wallet selects at most one descriptor document whose `wasm_hashes` contains
the resolved hash, according to its own source-priority policy. It MUST NOT
merge function descriptors from several documents.

#### 4.3 Validating a function descriptor

Before using a function descriptor, a wallet MUST load the SEP-48 spec from the
Wasm's `contractspecv0` section and check all of the following. If the section
is missing or does not decode, every function descriptor for that Wasm is
invalid.

- Exactly one `SC_SPEC_ENTRY_FUNCTION_V0` entry with the function's name
  exists. Zero or more than one is invalid.
- There is no unknown key anywhere inside the function descriptor, including
  inside field descriptors and `token` objects.
- `intent` and every `label` are non-empty and satisfy the charset requirement.
- Every `path` and every `token.path` resolves against the spec: its first
  element names an input of the function; each integer element fits in a signed
  32-bit value and steps into a `vec` type, a `tuple` type or a tuple struct
  (§2.1), and for a `tuple` or a tuple struct the index (negative values
  counting from the end) is within its arity; each string element steps into a
  member of a struct UDT declared in the spec that is not a tuple struct. No
  two fields have the same path.
- `auth`, if present, is a path that resolves against the spec in the same way
  and ends at spec type `address`.
- Every `format` is one of the defined values.
- `token` is present if and only if `format` is `amount`, and has exactly one
  of the shapes in §3.1.
- `decimals` is present if and only if `format` is `percent`, and is an integer
  from 0 to 38.
- `true` and `false` are both present if and only if `format` is `bool`, and
  are non-empty and satisfy the charset rule.
- `values` is present if and only if `format` is `enum`; it has at least one
  entry, its keys are decimal integer strings and its values are non-empty and
  satisfy the charset rule.
- Every field with a `format` has a path whose end type is in that format's
  accepted set, and every `token.path` ends at spec type `address`.

If any check fails, the function descriptor is invalid and the wallet renders
that function raw (§6). A failure in one function descriptor does not
invalidate other functions in the same document.

Wallets SHOULD cache validation results keyed by (descriptor document hash,
Wasm hash, function name); all three inputs are immutable. The descriptor
document hash is the SHA-256 of the exact UTF-8 bytes of the document as
published; any future attestation scheme SHOULD use the same definition.

#### 4.4 Argument check

This check applies to the host function invocation (§5.1), to authorization
nodes (§5.2) and to Stellar Asset Contract calls (§5.3).

1. The argument count MUST equal the number of spec inputs (for §5.3, the
   number of positional types).
1. Each field's path, each `token.path` and the `auth` path are followed
   through the argument values. An integer element requires an `SCV_VEC` (a
   `tuple` and a tuple struct are also encoded as `SCV_VEC`) and selects its
   element, counting negative values from the end. A string element requires an
   `SCV_MAP` and selects the entry whose key is the `SCV_SYMBOL` of that name.
   If the map has no entry for a member whose spec type is `option`, the
   member's value is taken to be `SCV_VOID`, because [CAP-86] lets a struct
   omit an empty member. A missing member of any other type is a check failure.
1. If a field's path steps into a `vec` at an index beyond its length, the
   field is omitted from the described view (it counts toward the omission
   indication of §5.1) and its token path is not evaluated. A `tuple` or a
   tuple struct that a path steps into MUST have exactly the number of elements
   its spec declares. Any other traversal failure, including one on a
   `token.path` or on the `auth` path, is a check failure.
1. The value a path ends at, for a field, a token reference or `auth`, MUST
   have the form below; for §5.3 a field's spec type is its positional type. A
   field whose path ends at any other spec type is not checked, nor is an
   argument that no path reaches.

| argument                                                      | required `SCVal`                                                                                                                                         |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| field of spec type `u32`, `i32`, `u64`, `i64`, `u128`, `i128` | `SCV_U32`, `SCV_I32`, `SCV_U64`, `SCV_I64`, `SCV_U128`, `SCV_I128` respectively                                                                          |
| field of spec type `timepoint`                                | `SCV_TIMEPOINT`                                                                                                                                          |
| field of spec type `duration`                                 | `SCV_DURATION`                                                                                                                                           |
| field of spec type `bool`                                     | `SCV_BOOL`                                                                                                                                               |
| field of spec type `address`                                  | `SCV_ADDRESS` with `SC_ADDRESS_TYPE_ACCOUNT` or `SC_ADDRESS_TYPE_CONTRACT`                                                                               |
| field of spec type `muxed_address`                            | `SCV_ADDRESS` with `SC_ADDRESS_TYPE_ACCOUNT`, `SC_ADDRESS_TYPE_CONTRACT`, `SC_ADDRESS_TYPE_MUXED_ACCOUNT` or `SC_ADDRESS_TYPE_MUXED_CONTRACT` ([CAP-84]) |
| end of a `token.path`                                         | `SCV_ADDRESS` with `SC_ADDRESS_TYPE_CONTRACT`                                                                                                            |
| end of the `auth` path                                        | `SCV_ADDRESS` with `SC_ADDRESS_TYPE_ACCOUNT` or `SC_ADDRESS_TYPE_CONTRACT`                                                                               |

Four conditions defined in §3 are check failures as well: values of `decimals`
that disagree, a `symbol` that breaks the character rule (§3.1), a `timestamp`
value with no RFC 3339 form (§3.2) and an `enum` value with no entry (§3.6).
Failing to obtain `decimals` or `symbol` is not a check failure; only the
affected field degrades (§3.1).

If the check fails, the whole invocation is rendered raw (§6) and the curated
intent MUST NOT be shown for it: such a call will fail on-chain, is
adversarial, cannot be shown as described, or (for an authorization node) does
not match the curator's `auth` assertion.

#### 4.5 Pre-validation for offline signers

Resolution (§4.1) and validation (§4.3) MAY be performed in advance by the
wallet vendor or by the descriptor source; the argument check (§4.4) MUST be
performed by the signer for each invocation. A signer that cannot read ledger
state obtains the contract-to-Wasm-hash mapping from a party it trusts and
inherits that party's trust (see [Limitations](#limitations)).

A pre-validated descriptor handed to such a signer MUST carry everything §4.4
and §5 need without the spec. Its format is the vendor's own (see
[Limitations](#limitations)).

### 5. Rendering

#### 5.1 Host function invocation

For an `INVOKE_HOST_FUNCTION` operation whose host function is
`HOST_FUNCTION_TYPE_INVOKE_CONTRACT`, the invocation is **described** when its
contract resolves (§4.1) to a Wasm hash with a valid function descriptor for
the invoked function (§4.3) and the argument check (§4.4) passes, or when §5.3
applies.

For a described invocation a wallet:

- MUST display the `intent` prominently, before the parameter list.
- MUST display the contract ID (full strkey viewable) and the function name.
- MUST display the listed fields, in descriptor order, using their `label` and
  `format`. Parameters not listed, and fields omitted under §4.4, are not shown
  in this described view.
- MUST keep a raw view of the whole invocation available on request: every
  argument in spec order, labelled with its spec name, rendered as in §3.0. A
  wallet SHOULD indicate in the described view when parameters were omitted
  (for example `+2 more parameters`), and MAY paginate on constrained displays.

A descriptor controls what the described view shows. It cannot make any part of
the invocation unviewable, and it does not change what is signed.

Host functions other than `HOST_FUNCTION_TYPE_INVOKE_CONTRACT` (contract
creation, Wasm upload) are out of scope of this SEP, as are transaction-level
fields (fee, sequence number, memo, preconditions) and classic operations,
which a wallet displays as it already does.

#### 5.2 Authorization entries

Each `SorobanAuthorizationEntry` carries a `SorobanAuthorizedInvocation` tree
that the signer is authorizing. Its nodes need not repeat the host function's
arguments: a node's argument list is whatever the contract demanded
authorization for (§2), and sub-invocations authorize calls the signer never
issued directly. Every node of every entry is evaluated independently, except
under the legacy rule below.

**Which nodes are described.** A `SOROBAN_AUTHORIZED_FUNCTION_TYPE_CONTRACT_FN`
node is **described** when all of these hold:

- Its function has an `auth` path. Either its contract is a Stellar Asset
  Contract as established by §5.3 and its function is listed there, or its
  contract resolves (§4.1) to a Wasm hash with a valid function descriptor for
  the node's function name that sets `auth`.
- The argument check (§4.4) passes against the node's arguments (against §5.3's
  positional types for a Stellar Asset Contract).
- The address at the `auth` path is the entry's authorizer, as defined under
  Credentials below.

A described node is rendered as in §5.1 or §5.3, with `self` in §3.1 meaning
the node's `contractAddress`. Every other node, including nodes of other
function types, is rendered raw: contract ID (or creation arguments), function
name, each argument raw, with no spec-derived labels. §6 applies to those
nodes.

**Credentials.** For each displayed entry (see the omission rule below), the
entry's authorizer and what a wallet does depend on the entry's credentials:

| credentials                                                                    | the entry's authorizer                                                                                                                                                                       | what the wallet does                                                                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SOROBAN_CREDENTIALS_SOURCE_ACCOUNT`                                           | The operation's source account (the transaction's source if the operation sets none; for a fee-bump transaction, the inner transaction's). A muxed account counts as its underlying account. | MUST indicate that this account authorizes.                                                                                                                                                                                                                                       |
| `SOROBAN_CREDENTIALS_ADDRESS_V2`, `SOROBAN_CREDENTIALS_ADDRESS_WITH_DELEGATES` | The credentials' `address`.                                                                                                                                                                  | MUST display the `address` and `signatureExpirationLedger`. These credentials bind the address in the signature payload ([CAP-71]).                                                                                                                                               |
| `SOROBAN_CREDENTIALS_ADDRESS` (legacy)                                         | The credentials' `address`.                                                                                                                                                                  | MUST display the `address` and `signatureExpirationLedger`. The signature payload (`ENVELOPE_TYPE_SOROBAN_AUTHORIZATION`) does not contain the address, so the wallet SHOULD indicate that the signature does not cover it. Nodes are described only under the legacy rule below. |
| any other type                                                                 | None.                                                                                                                                                                                        | Not covered by this version. The wallet renders the entry's nodes raw under §6 regardless of any descriptor.                                                                                                                                                                      |

**Legacy credentials.** In an entry with `SOROBAN_CREDENTIALS_ADDRESS`
credentials, a node is described only if the entry's root node is described.
Otherwise every node of the entry is rendered raw under §6, including Stellar
Asset Contract calls. The sub-invocations of an entry are permissions that need
not all be used, so only the root node's `auth` argument, which is part of the
signed invocation, ties the signature to one address.

**Signing a single entry.** A wallet is sometimes asked to sign a single
authorization entry rather than a whole transaction, for example when a dApp
collects the user's authorization through SEP-43 `signAuthEntry` and submits
the transaction itself. What is signed is then a bare `HashIDPreimage`:

| preimage type                                      | what the wallet does                                                                                                                                                                                   |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ENVELOPE_TYPE_SOROBAN_AUTHORIZATION_WITH_ADDRESS` | Renders the invocation tree it contains under this section, with the preimage's `address` as the entry's authorizer, and MUST additionally display that `address` and the `signatureExpirationLedger`. |
| `ENVELOPE_TYPE_SOROBAN_AUTHORIZATION` (legacy)     | Out of scope: the preimage contains no address. The wallet renders every node raw under §6 regardless of any descriptor.                                                                               |
| any other type                                     | Not covered by this version.                                                                                                                                                                           |

When the displayed `address` is not the signer's own account, for example
because the signer is a delegate under
`SOROBAN_CREDENTIALS_ADDRESS_WITH_DELEGATES`, the wallet SHOULD make clear that
the signature authorizes on behalf of that address. An address supplied by the
requester (e.g. SEP-43 `opts.address`) MUST NOT be presented as the authorizer.

**Tree structure.** The tree structure (which invocations are sub-invocations
of which) MUST be shown. When the enclosing operation is available, a wallet
MAY omit an entry when all of these hold:

- its credentials are `SOROBAN_CREDENTIALS_SOURCE_ACCOUNT`;
- the operation sets no source account of its own (so the authorizer is the
  transaction source the user already sees);
- `rootInvocation.function` is `SOROBAN_AUTHORIZED_FUNCTION_TYPE_CONTRACT_FN`
  with `contractFn` XDR-identical to the operation's
  `hostFunction.invokeContract`;
- `rootInvocation.subInvocations` is empty.

#### 5.3 Stellar Asset Contracts

Stellar Asset Contracts have no Wasm and no `contractspecv0` section, so
descriptors cannot describe them. A contract is treated as a Stellar Asset
Contract only if the wallet holds an `Asset` that hashes to its contract ID, as
described below; a host's assertion alone never suffices, and otherwise §6
applies.

The interface of a Stellar Asset Contract is fixed by the protocol, so a wallet
renders `transfer`, `approve`, `burn` and `trust` using the following
**built-in function descriptors**, both as host function invocations and as
authorization nodes (each function calls `require_auth()` on its first argument
and demands no other authorization, so `auth` names that argument). They are
part of the wallet, not a published document, so they form a `functions` object
only and are not subject to §1. Because a SAC has no spec, path names cannot be
resolved; the names below are display names and bind to argument **positions**:

| function   | position 0 | position 1 | position 2 | position 3          |
| ---------- | ---------- | ---------- | ---------- | ------------------- |
| `transfer` | `from`     | `to`       | `amount`   | (none)              |
| `approve`  | `from`     | `spender`  | `amount`   | `live_until_ledger` |
| `burn`     | `from`     | `amount`   | (none)     | (none)              |
| `trust`    | `addr`     | (none)     | (none)     | (none)              |

with positional types `address, muxed_address, i128` for `transfer`,
`address, address, i128, u32` for `approve`, `address, i128` for `burn`, and
`address` for `trust`.

```json
{
  "transfer": {
    "intent": "Send tokens",
    "auth": ["from"],
    "fields": [
      {
        "path": ["amount"],
        "label": "Amount",
        "format": "amount",
        "token": { "self": true }
      },
      { "path": ["to"], "label": "Recipient" },
      { "path": ["from"], "label": "From" }
    ]
  },
  "approve": {
    "intent": "Approve token spending",
    "auth": ["from"],
    "fields": [
      { "path": ["spender"], "label": "Spender" },
      {
        "path": ["amount"],
        "label": "Allowance",
        "format": "amount",
        "token": { "self": true }
      },
      { "path": ["live_until_ledger"], "label": "Expires (ledger)" },
      { "path": ["from"], "label": "Owner" }
    ]
  },
  "burn": {
    "intent": "Burn tokens",
    "auth": ["from"],
    "fields": [
      {
        "path": ["amount"],
        "label": "Amount",
        "format": "amount",
        "token": { "self": true }
      },
      { "path": ["from"], "label": "From" }
    ]
  },
  "trust": {
    "intent": "Add trustline",
    "auth": ["addr"],
    "fields": [{ "path": ["addr"], "label": "Account" }]
  }
}
```

The `transfer`, `approve` and `burn` entries are deliberately identical in
shape to those in the SEP-41 token example under
[Example Usage](#example-usage), so a wallet can use one rendering path for
both: for a Wasm token the names resolve through the spec, for a SAC they
resolve through the position table above. `trust` exists only on Stellar Asset
Contracts ([CAP-73]): it creates an unlimited trustline to the contract's asset
for the given account.

For a Stellar Asset Contract, `decimals` is 7 and the displayed symbol is the
asset code, or `XLM` for the native asset (whose contract reports `native`).
The wallet MUST obtain the `Asset` from a source it trusts or from the host,
and MUST verify that the asset's contract ID (SHA-256 of a `HashIDPreimage` of
type `ENVELOPE_TYPE_CONTRACT_ID` with `CONTRACT_ID_PREIMAGE_FROM_ASSET` and the
network ID) equals the contract's ID before using its code and issuer. The
issuer SHOULD be viewable. A `trust` call has no amount to carry the symbol, so
for `trust` the wallet MUST display the asset code next to the contract ID.
Symbol and issuer MUST NOT come from any other source; in particular a host
MUST NOT be able to relabel the native asset.

> **TODO (pre-submission draft):** an issued asset can reuse any code,
> including `XLM` or `USDC`, and its Stellar Asset Contract passes the check
> above, so `1 XLM` on screen need not be the native asset. Decide whether the
> issuer of a non-native asset should be displayed together with its code (a
> MUST) instead of only being viewable.

Other Stellar Asset Contract functions are rendered raw under §6.

An `approve` with an amount of `0` revokes the allowance, and SEP-41 then lets
`live_until_ledger` be `0` as well. It is rendered with the same descriptor, as
`Approve token spending` with an allowance of `0`: an `intent` does not vary
with values in this version. The same holds for Wasm SEP-41 tokens.

> **TODO (pre-submission draft):** survey which other SAC functions users
> actually sign and whether they need a built-in rendering. Candidates noted so
> far: `transfer_from`/`burn_from` (spender-authorized, usually by a contract),
> and the issuer-only functions `mint`, `clawback`, `set_admin`,
> `set_authorized`.

### 6. Fallback

When an invocation or an authorization node is not described, a wallet:

- MUST display the contract ID, the function name, and every argument (for a
  contract-creation node, its creation arguments).
- MAY label arguments with names from the SEP-48 spec for the host function
  invocation, but MUST NOT do so for authorization nodes (their arguments need
  not be the function's declared parameters).
- MUST indicate to the user that the call is not described, for example
  "Unverified contract call" or "No description available for this call". The
  exact wording is not mandated. A wallet MAY add the reason, for example that
  no descriptor exists for the contract or that a token's metadata sources
  disagree.
- MUST NOT display an `intent` or `label` taken from a descriptor that did not
  pass §4 for this Wasm hash, including descriptors matched by contract address
  or by function name alone.

### 7. Distribution and trust

This SEP defines the descriptor format and how it is validated and rendered. It
does not define where descriptors are published or how a wallet decides to
trust a source. Possible venues include:

- A community-maintained public repository of descriptor files, reviewed before
  merge, mirrored by wallets.
- Descriptors bundled with a wallet release.
- Vendor-built, per-network snapshots for offline signers that map contract IDs
  to pre-validated descriptors (§4.5).
- A future on-chain registry or a future SEP-46 meta key pointing to a
  descriptor.

A wallet MUST treat a descriptor as an assertion by whoever curated the source
it came from, not as a property of the contract. A descriptor that passes §4 is
consistent with the contract's declared interface. That does not mean the
contract is honest.

## Example Usage

### SEP-41 token (Wasm implementation)

The Aquarius liquidity pool share token on the public network (one instance is
`CDXRIV6XHJWJXCFCP7CGPPQCLQSVMJ64UOOJY2XLCFAOMF6OUFEH2VDD`; Wasm
`72eb5d52c532e78ccc6c8a7f8523c458e03764074717172f69c2539e754ce624`, as deployed
on 2026-11-04) is a SEP-41 token whose `transfer`, `approve` and `burn` each
call `from.require_auth()`. Parameter names vary between implementations even
when the interface is the same: this contract names the fourth `approve` input
`expiration_ledger` (an older naming), while [SEP-41], the current soroban-sdk
`TokenInterface` and the soroban-examples token use `live_until_ledger`, and
the OpenZeppelin fungible token names the first input `owner` rather than
`from`. A descriptor MUST use the names the Wasm's own spec declares, which is
why one descriptor cannot be shared across token implementations by name.

```json
{
  "version": 1,
  "wasm_hashes": [
    "72eb5d52c532e78ccc6c8a7f8523c458e03764074717172f69c2539e754ce624"
  ],
  "functions": {
    "transfer": {
      "intent": "Send tokens",
      "auth": ["from"],
      "fields": [
        {
          "path": ["amount"],
          "label": "Amount",
          "format": "amount",
          "token": { "self": true }
        },
        { "path": ["to"], "label": "Recipient" },
        { "path": ["from"], "label": "From" }
      ]
    },
    "approve": {
      "intent": "Approve token spending",
      "auth": ["from"],
      "fields": [
        { "path": ["spender"], "label": "Spender" },
        {
          "path": ["amount"],
          "label": "Allowance",
          "format": "amount",
          "token": { "self": true }
        },
        { "path": ["expiration_ledger"], "label": "Expires (ledger)" },
        { "path": ["from"], "label": "Owner" }
      ]
    },
    "burn": {
      "intent": "Burn tokens",
      "auth": ["from"],
      "fields": [
        {
          "path": ["amount"],
          "label": "Amount",
          "format": "amount",
          "token": { "self": true }
        },
        { "path": ["from"], "label": "From" }
      ]
    }
  }
}
```

A wallet rendering `approve(from, spender, 1000000000, 65623765)` on an
instance whose metadata resolves to 7 decimals and the symbol `EXT` might show:

```
Approve token spending

Contract          CDLZ…7XQJ (full ID viewable)
Function          approve
Spender           GB7XW…K2QF (full strkey viewable)
Allowance         100 EXT
Expires (ledger)  65623765
Owner             GA3RT…9MPL
```

### Aquarius AMM (mainnet)

The Aquarius AMM on the public network, as deployed on 2026-11-02, has two
relevant contracts. Users invoke the swap front contract
`CCPGFQUTSEHDIQODRE3GJDNE64A35HZ32L7LPDN7GXOCIYNBJSMS6V6B` (Wasm
`131db85e62b0f7e3683bf4b5a6ae22b801d9acb2effd5cf3d78e9921b77b7c94`), whose
`contractspecv0` section declares

```rust
fn swap_chained(user: Address, swaps_chain: Vec<(Vec<Address>, BytesN<32>, Address)>,
                token_in: Address, in_amount: u128, out_min: u128, fee_fraction: u32) -> u128;
```

and which calls `user.require_auth()`. The output token is the third member of
the last `swaps_chain` tuple, which a token path reaches:

```json
{
  "version": 1,
  "wasm_hashes": [
    "131db85e62b0f7e3683bf4b5a6ae22b801d9acb2effd5cf3d78e9921b77b7c94"
  ],
  "functions": {
    "swap_chained": {
      "intent": "Swap tokens",
      "auth": ["user"],
      "fields": [
        {
          "path": ["in_amount"],
          "label": "You pay",
          "format": "amount",
          "token": { "path": ["token_in"] }
        },
        {
          "path": ["out_min"],
          "label": "Min. received",
          "format": "amount",
          "token": { "path": ["swaps_chain", -1, 2] }
        },
        { "path": ["user"], "label": "Account" },
        {
          "path": ["fee_fraction"],
          "label": "Fee",
          "format": "percent",
          "decimals": 4
        }
      ]
    }
  }
}
```

`token_in` and `swaps_chain` are not listed as fields, so they appear only in
the raw view, but the token paths of `in_amount` and `out_min` read them.
`fee_fraction` uses `percent` with `decimals: 4`, following the Aquarius
convention of a 10,000 denominator for fee fractions.

The front contract delegates to the liquidity pool router
`CBQDHNBFBZYE4MKPWBSJOPIYLW4SFSXAXUTSXJN76GNKYVYPCKWC6QUK` (Wasm
`c99539b023df4a28d7857d33ca4c915c2e20fec04f12b125132b84fbdf94dad3`), whose
single-pool `swap` also calls `user.require_auth()` and names both tokens as
top-level parameters, so it can be described fully:

```rust
fn swap(user: Address, tokens: Vec<Address>, token_in: Address, token_out: Address,
        pool_index: BytesN<32>, in_amount: u128, out_min: u128) -> u128;
```

```json
{
  "version": 1,
  "wasm_hashes": [
    "c99539b023df4a28d7857d33ca4c915c2e20fec04f12b125132b84fbdf94dad3"
  ],
  "functions": {
    "swap": {
      "intent": "Swap tokens",
      "auth": ["user"],
      "fields": [
        {
          "path": ["in_amount"],
          "label": "You pay",
          "format": "amount",
          "token": { "path": ["token_in"] }
        },
        {
          "path": ["out_min"],
          "label": "You receive (min)",
          "format": "amount",
          "token": { "path": ["token_out"] }
        },
        { "path": ["user"], "label": "Account" }
      ]
    }
  }
}
```

A router whose hop list is a plain `Vec<Address>` of tokens would instead
reference `{ "path": ["path", 0] }` and `{ "path": ["path", -1] }`.

### Authorization tree

Mainnet transaction
`02c785c9a74a500a78025552cf44766a58885236aba2396a456d9651c3e08a44` is a
fee-bump transaction whose inner transaction, with source `GDM22NKY…5DHK`,
invokes the Aquarius front contract:

```
swap_chained(GDM22NKY…5DHK, [[[CAS3J7GY…XOWMA, …], <pool hash>, …]],
             CAS3J7GY…XOWMA, 10000000, 27796236, 0)
```

`CAS3J7GY…XOWMA` is the Stellar Asset Contract of the native asset; the last
hop's output token, `CBLLEW7H…CIZA`, is the Stellar Asset Contract of `XRF`.
The operation carries one authorization entry with
`SOROBAN_CREDENTIALS_SOURCE_ACCOUNT` credentials whose root is that same
`swap_chained` call and whose single sub-invocation is
`CAS3J7GY…XOWMA.transfer(GDM22NKY…5DHK, CCPGFQUT…6V6B, 10000000)`. The front
contract's own call into the router does not appear in the tree because the
front contract, not the user, is its invoker.

With the descriptor above, a wallet renders:

```
Swap tokens

Contract            CCPGFQUT…6V6B (full ID viewable)
Function            swap_chained
You pay             1 XLM
Min. received       2.7796236 XRF
Account             GDM22NKY…5DHK
Fee                 0%
+2 more parameters  (token_in and swaps_chain, in the raw view)

Authorized by the transaction source GDM22NKY…5DHK
  swap_chained(…)                      ← same call as above
  └ Send tokens                        ← §5.3
      Contract   CAS3J7GY…XOWMA (XLM)
      Function   transfer
      Amount     1 XLM
      Recipient  CCPGFQUT…6V6B
      From       GDM22NKY…5DHK
```

The entry is not omittable under §5.2 because it has a sub-invocation. Both
nodes are described because the address at their `auth` path, `user` for
`swap_chained` and `from` for `transfer`, is the entry's authorizer, the inner
transaction's source. The fee-bump's fee source is not shown as an authorizer;
the inner transaction's source is. The `1 XLM` on both lines comes from §5.3's
asset check (the asset hashes to `CAS3J7GY…XOWMA`), not from anything the host
asserted.

Much of the practical coverage of this SEP comes from this pattern: even where
a protocol's own function is undescribed, the token `transfer` sub-invocation
in its authorization tree states exactly what leaves the user's account. With
legacy credentials this holds only when the root function is described as well
(§5.2).

## Limitations

This first version gives up some coverage to stay simple. The limitations below
are known and are accepted for this version. Several are expected to be
addressed in a later version or in a separate SEP.

### Scope of what can be described

- **Intent only.** A descriptor states what a call is meant to do. It cannot
  state what will actually happen, such as which branch executes, how much a
  swap really returns, or whether a "deposit" contract is honest. Effect
  previews derived from `simulateTransaction` events and state changes are a
  separate concern and are not covered here.
- **Paths reach fixed positions only.** A path can step into vectors, tuples
  and struct members, but not into `map` or `option` values, and it names one
  position at a time. To describe every element of a variable-length vector, a
  curator has to list each position up to a maximum of their choosing. A
  lending pool's `Vec<Request>` with up to four requests takes four sets of
  fields, for example. Elements beyond the maximum appear only in the raw view.
  A per-element construct like ERC-7730's `[]` is deferred until
  variable-length arguments prove common enough to justify it.
- **The token must be in the arguments.** An amount cannot use the `amount`
  format when its token is determined by contract instance state. Examples are
  a single-token staking contract that stores its token at construction, and a
  pool whose `swap` takes indices into its own token list. In practice the
  `transfer` sub-invocation in the authorization tree shows the real token and
  amount.
- **English only.** `intent` and `label` are ASCII English and there is no
  localization mechanism. This is a cost for users who do not read English (see
  Design Rationale). Intent identifiers are an open item (see the TODO in §2).
- **No conditional display.** A field is either listed and shown, or not listed
  and visible only in the raw view. There is no way to leave a field to the
  wallet's discretion or to show it only under some condition, as ERC-7730's
  `visible: optional` and `ifNotIn` allow. Such rules are deferred.

### Binding and coverage

- **One descriptor for every instance of a Wasm.** A descriptor applies to
  every instance of the Wasm and cannot tell instances apart, even when they
  differ in ways that matter, such as two pools of a factory, or a legitimate
  and a scam deployment of the same token code. Descriptors carry no contract
  name and no fixed contract ID because a curator cannot vouch for instances it
  never saw. Users have to rely on the contract ID and on resolved token
  symbols to tell instances apart.
- **Stellar Asset Contracts: `transfer`, `approve`, `burn` and `trust` only.**
  Other SAC functions (`transfer_from`, `burn_from`, `mint`, `clawback`,
  `set_admin`, `set_authorized`) are rendered raw with the §6 marker (see the
  TODO in §5.3).
- **Every Wasm upgrade needs a new descriptor entry.** Binding to the Wasm hash
  is what makes a descriptor trustworthy, but it also means that an upgraded
  contract, or a CAP-85 fleet whose owner switched its executable, is
  undescribed until someone publishes a descriptor that lists the new hash.
- **Interface-level matching is not supported.** A descriptor cannot say "this
  applies to any contract whose spec implements SEP-41". Each Wasm must be
  listed explicitly.
- **Offline signers rely on a snapshot.** A signer that cannot read ledger
  state cannot resolve a contract ID to a Wasm hash. It relies on a
  vendor-built, per-network snapshot mapping contract IDs to pre-validated
  descriptors. That snapshot is the trust boundary and goes stale when a
  contract upgrades or a CAP-85 owner switches its fleet. Stellar Asset
  Contracts are the exception, because §5.3 can be verified offline from the
  asset.

### Authorization tree

- **`auth` is asserted by the curator and only partly checked.** A wallet
  checks the argument count, the types of the listed fields and the address at
  the `auth` path. It cannot detect a function that uses
  `require_auth_for_args()` with a list that keeps the authorizer in the same
  position and has the same length and types as the function's own arguments,
  but different meanings. If a curator wrongly gives such a function `auth`,
  its authorization nodes are rendered with the wrong labels. The only defence
  is the curator's source review required in §2.
- **One authorizer, and it must be an argument.** A function whose authorizer
  is read from contract state, such as an administrator, or that demands
  authorization from several addresses, cannot set `auth`, so its authorization
  nodes are raw. Its host function invocation is still described.
- **Functions without `auth` show raw authorization nodes.** This is the safe
  behaviour. It does mean that a protocol whose host function invocation is
  described may still show an undescribed authorization entry until a curator
  sets `auth`.
- **No descriptor for `require_auth_for_args` shapes.** A later version may
  allow a function descriptor to declare the argument list its authorization
  uses so those nodes can be labelled.
- **Legacy credentials are described only from the root.** Most address
  credentials on the public network are still of the legacy kind. An entry with
  legacy credentials is described only when its root node is described, so a
  token `transfer` nested under an undescribed function stays raw, and a legacy
  entry signed on its own is always raw (§5.2). Entries with the address-bound
  credentials of [CAP-71] do not have this restriction.
- **Minimal credential display.** This SEP requires the authorizer and
  expiration to be shown but does not specify how to present nonces, delegated
  signer sets, or custom-account credentials beyond that.
- **Other credential types are not described.** Entries with credentials beyond
  those named in §5.2 are rendered raw, and other preimage types are not
  covered. [CAP-90] proposes credentials with a custom payload whose signed
  preimage carries only a hash of the address-bound preimage. A signer shown
  that preimage alone cannot see the invocation tree unless the host also
  supplies the inner preimage to be hashed and compared. How such entries are
  delivered and displayed is left to a later version.

[CAP-90]:
  https://github.com/stellar/stellar-protocol/blob/master/core/cap-0090.md

### Trust and distribution

- **No registry, no signing, no attestation.** Descriptors are plain, unsigned
  JSON, so whether one is trustworthy depends entirely on where the wallet got
  it (§7 and Security Concerns).
- **No revocation mechanism.** There is no way to revoke or supersede a
  published descriptor other than the source removing it.
- **No hardware-wallet transport encoding.** How a host transfers a descriptor
  (or a vendor-signed derivative of it) to a constrained signer, and how the
  signer verifies it, is out of scope. Vendors are expected to compile
  descriptors into their own signed formats until a shared encoding is
  standardized.
- **Token metadata depends on the wallet.** Offline signers can only resolve
  `decimals` and `symbol` from a curated list, so an amount for an unlisted
  token renders as a raw integer on such devices even when a descriptor is
  present.

### Dependence on SEP-48

- **The spec is self-reported and not validated.** Stellar Core does not check
  that the `contractspecv0` section matches the Wasm's exports. If a Wasm has
  an incorrect or truncated spec, descriptors for it usually fail validation,
  which is safe. If the spec is wrong in a way that happens to agree with the
  descriptor, the call is rendered with wrong labels. Curators who publish a
  descriptor are expected to check the spec against the source.

## Design Rationale

**Why an off-chain descriptor instead of extending SEP-48 `doc` or adding a
SEP-46 meta key?** There are three reasons. First, anything embedded in the
Wasm is written by the contract author, and a malicious author can write a
reassuring description. The EVM's NatSpec and EIP-3224 experiments were
abandoned for this reason. A descriptor kept outside the Wasm can be curated by
parties other than the author. Second, embedding does not work for code that is
already deployed, so thousands of existing Wasms would remain undescribed.
Third, hardware wallets need the descriptor delivered and verified separately
from the Wasm in any case, so putting it in the Wasm does not help them. A
future SEP-46 key may still be useful as a hint that points to a descriptor.

**Why bind to the Wasm hash rather than the contract address, and why validate
against SEP-48?** The interface spec belongs to the Wasm, so a descriptor can
only be validated against a Wasm hash. That validation is cheap. A descriptor
that names a parameter the function does not have, or applies `amount` to an
`address`, is either a mistake or an attack, and the wallet should not render
it. The hash is also immutable and the same on every network. An address
differs per network and can be upgraded to new code, so a descriptor keyed by
address would silently describe the wrong code after an upgrade. ERC-7730 is
still working through this proxy-upgrade problem. Offline signers that cannot
resolve the hash themselves still benefit. Curation happens once per Wasm, a
vendor expands it to contract IDs mechanically, and a stale entry can be
detected by resolving each contract ID again, which an address-keyed list does
not allow. The price is that instances cannot be told apart, as described under
Limitations.

**Why so few formats?** A format is included only when showing the raw value of
a parameter would mislead the user. `amount`, `timestamp`, `duration`,
`percent`, `bool` and `enum` meet that criterion for the call shapes surveyed
so far. The set is expected to grow, by the same criterion, as the survey in
the §3 TODO is completed.

**Why does `auth` name the authorizing argument?** Inferring the authorization
shape from the argument count and types fails on the first real case, the one
described in §2, where the count and types are the same but the meaning
differs. A yes-or-no assertion by the curator would fix the labels, but a
wallet could not check it. Naming the argument gives the wallet something to
check. It compares the address at that path with the entry's authorizer, which
catches the §2 case, because there the first argument was a token contract and
not the signer. It also puts the authorizer inside the signed invocation, which
is what allows an entry with legacy credentials to be described at all (§5.2).
In a sample of authorization entries from the public network (November 2026),
the authorizer was one of the node's top-level arguments in 2,783 of 2,786
nodes. The other three were operator calls that repeat the transaction's main
invocation, which §5.1 describes without `auth`. The key is optional, so a
curator who does nothing gets the safe outcome.

**Why absolute paths and no per-element loop?** A path is the smallest
extension of "name an input" that reaches the positions real contracts use,
such as a token at a fixed place in a tuple or a member of a struct argument. A
loop over vector elements would add relative paths, nesting rules and an
unbounded number of displayed rows. Listing positions one by one makes
descriptors longer but leaves the rendering model unchanged.

**Why must the raw view always be available?** The Bybit incident
(February 2025) showed that the field that matters can be one the interface
does not highlight (`operation = 1`, a delegatecall), and that an interface can
be compromised. A descriptor decides what the described view shows, so that
common calls are easy to read on a small screen, but it can never make a
parameter unviewable. The full raw invocation stays one step away, and the
described view says when something was left out. The guarantee is that every
value can be checked, even though not every value is shown by default.

**Why does a single failed check make the whole invocation raw?** A described
view tells the user that the descriptor fits this call. If one value
contradicts the descriptor or cannot be shown as described, that is no longer
true, so the wallet shows the whole call raw instead of a description that is
wrong in one line. One rule for every such case is also easier for wallets to
implement the same way. Missing token metadata does not contradict anything, so
in that case only the amount is shown in raw units and the rest of the
description stands.

**Why a built-in rule for Stellar Asset Contracts?** SAC `transfer` is probably
the most common Soroban call that users sign, and it is the deepest node of
nearly every swap or deposit authorization tree. SACs have no Wasm and no spec,
so a descriptor cannot cover them. Marking every XLM or USDC transfer
"unverified" would teach users to ignore the marker. The SAC interface is fixed
by the protocol and its asset can be verified offline by hashing, which is what
makes a built-in rule possible. Trezor firmware 2.12.5 already renders
`transfer` and `approve` as token operations. `burn` is included because a
sample of mainnet traffic showed SAC `burn` to be the most frequent
authorization node of all, produced by staking flows that burn a token through
its SAC. `trust` is included because a new holder signs it to open a trustline
from a contract call, as in the SEP-61 onboarding flow. It had not yet appeared
in that sample.

**Why English only?** Localized intents would multiply the curation work and
complicate hardware-wallet storage. They would also need a review process for
each language, so that a translation cannot say something the English does not.
ERC-7730 made the same choice. Wallets are free to translate `label` strings
themselves, but `intent` is the phrase the curator vouched for.

**Why not require a registry in this version?** The format has to be agreed
first, and it does not depend on where descriptors are hosted. ERC-7730 shipped
as a format with a single-vendor registry, gained adoption, and only then moved
to neutral stewardship with an attestation layer. Specifying how a registry is
governed before anyone has published a descriptor would hold the format back.

## Security Concerns

- **False or compromised descriptors.** Validation checks that a descriptor is
  consistent with the contract's interface. It cannot check that the descriptor
  is truthful. A curated source is the only defence, and a compromised source
  is as bad as a compromised wallet UI. Wallets SHOULD obtain descriptors from
  sources with review processes and SHOULD consider agreement between
  independent sources before rendering an intent, particularly on hardware
  wallets where the rendered text is the user's only view of the transaction.
- **A described call is not necessarily safe.** A correct descriptor on a
  malicious instance renders a truthful `Send tokens` / `100 SCAM`. Descriptors
  do not vouch for the contract, the token, or the counterparty. Wallets MUST
  keep displaying the contract ID and SHOULD combine descriptors with whatever
  instance-level reputation or risk signals they already use.
- **Code can change between signing and execution.** Neither a transaction nor
  an authorization entry commits to a Wasm hash. If a contract is upgraded, or
  a CAP-85 owner switches its fleet, between signing and execution, the code
  that runs is not the code that was described, because a descriptor describes
  the Wasm resolved at signing time. Long `signatureExpirationLedger` values
  widen this window, so wallets SHOULD prefer short expirations and SHOULD warn
  on unusually long ones.
- **Legacy credentials.** The signature payload of
  `SOROBAN_CREDENTIALS_ADDRESS` does not contain the authorizing address. When
  such an entry is described, the link between the signature and the displayed
  authorizer rests on the `auth` assertion for the root function, not on the
  protocol. If that assertion is wrong, or the contract's code changes before
  execution, the signature could be used for another account that accepts the
  same signing key. The address-bound credentials of [CAP-71] do not have this
  weakness.
- **Token metadata spoofing.** A token contract can return any `symbol` and any
  `decimals`, and can return different values to a simulation than at
  execution. An `amount` rendered as `100 USDC` using metadata read from the
  contract itself is only as trustworthy as the contract. A hostile `decimals`
  of `2` turns `1000000000` into `10,000,000.00`, and a `decimals` of `18`
  turns a large transfer into `0.000000001`. A hostile `symbol` can carry
  control characters or be long enough to push other fields off a small screen.
  §3.1 therefore requires contract-sourced metadata to be marked unverified,
  symbols with unsafe characters to be rejected and long ones truncated, and
  unresolved amounts to be marked as raw units. Wallets SHOULD prefer curated
  token lists. For Stellar Asset Contracts, §5.3 requires the asset to hash to
  the contract ID so that a host cannot relabel one asset as another.
- **Address substitution.** Labels from an address book or name service can be
  poisoned. The requirement to keep the full strkey viewable exists so a user
  can verify the counterparty independently of any label.

## Appendix A: JSON Schema (informative)

The following JSON Schema (draft 2020-12) captures the structural rules of §1,
§2 and §3. It is informative: §4.3 remains the normative validation, since the
schema cannot check rules that depend on the contract's spec (parameter
existence and types) or that JSON Schema cannot express (no two fields with the
same path, no duplicate object keys).

The name limits in the schema come from the XDR of [SEP-48]: a function name is
an `SCSymbol`, at most 32 characters from `[A-Za-z0-9_]`, and the name of an
input or of a struct member is a string of at most 30 bytes.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Contract Display Descriptor document (v1)",
  "type": "object",
  "required": ["version", "wasm_hashes", "functions"],
  "properties": {
    "version": { "const": 1 },
    "wasm_hashes": {
      "type": "array",
      "minItems": 1,
      "items": { "type": "string", "pattern": "^[0-9a-f]{64}$" }
    },
    "functions": {
      "type": "object",
      "propertyNames": { "pattern": "^[A-Za-z0-9_]*$", "maxLength": 32 },
      "additionalProperties": { "$ref": "#/$defs/function" }
    }
  },
  "$defs": {
    "ascii": { "type": "string", "pattern": "^[\\x20-\\x7E]*$" },
    "path": {
      "type": "array",
      "minItems": 1,
      "prefixItems": [{ "type": "string", "maxLength": 30 }],
      "items": {
        "oneOf": [
          { "type": "integer", "minimum": -2147483648, "maximum": 2147483647 },
          { "type": "string", "maxLength": 30 }
        ]
      }
    },
    "function": {
      "type": "object",
      "required": ["intent"],
      "additionalProperties": false,
      "properties": {
        "intent": { "$ref": "#/$defs/ascii", "minLength": 1 },
        "auth": { "$ref": "#/$defs/path" },
        "fields": { "type": "array", "items": { "$ref": "#/$defs/field" } }
      }
    },
    "field": {
      "type": "object",
      "required": ["path", "label"],
      "additionalProperties": false,
      "properties": {
        "path": { "$ref": "#/$defs/path" },
        "label": { "$ref": "#/$defs/ascii", "minLength": 1 },
        "format": {
          "enum": [
            "amount",
            "timestamp",
            "duration",
            "percent",
            "bool",
            "enum"
          ]
        },
        "token": { "$ref": "#/$defs/token" },
        "decimals": { "type": "integer", "minimum": 0, "maximum": 38 },
        "true": { "$ref": "#/$defs/ascii", "minLength": 1 },
        "false": { "$ref": "#/$defs/ascii", "minLength": 1 },
        "values": {
          "type": "object",
          "minProperties": 1,
          "propertyNames": { "pattern": "^-?[0-9]+$" },
          "additionalProperties": { "$ref": "#/$defs/ascii", "minLength": 1 }
        }
      },
      "allOf": [
        {
          "if": {
            "properties": { "format": { "const": "amount" } },
            "required": ["format"]
          },
          "then": { "required": ["token"] },
          "else": { "not": { "required": ["token"] } }
        },
        {
          "if": {
            "properties": { "format": { "const": "percent" } },
            "required": ["format"]
          },
          "then": { "required": ["decimals"] },
          "else": { "not": { "required": ["decimals"] } }
        },
        {
          "if": {
            "properties": { "format": { "const": "bool" } },
            "required": ["format"]
          },
          "then": { "required": ["true", "false"] },
          "else": {
            "allOf": [
              { "not": { "required": ["true"] } },
              { "not": { "required": ["false"] } }
            ]
          }
        },
        {
          "if": {
            "properties": { "format": { "const": "enum" } },
            "required": ["format"]
          },
          "then": { "required": ["values"] },
          "else": { "not": { "required": ["values"] } }
        }
      ]
    },
    "token": {
      "oneOf": [
        {
          "type": "object",
          "additionalProperties": false,
          "required": ["self"],
          "properties": { "self": { "const": true } }
        },
        {
          "type": "object",
          "additionalProperties": false,
          "required": ["path"],
          "properties": { "path": { "$ref": "#/$defs/path" } }
        }
      ]
    }
  }
}
```

## Changelog

- `v0.0.1` - Initial draft.
