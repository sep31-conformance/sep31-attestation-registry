# sep31-attestation-registry

A minimal Soroban smart contract that stores on-chain attestations of
[SEP-31](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0031.md)
corridor conformance checks — a durable, publicly queryable answer to "is
this receiving anchor's SEP-31 corridor currently verified?" that doesn't
depend on trusting whoever runs the checker.

Part of a project mirroring [`sep24-attestation-registry`](https://github.com/SEP-24-conform/sep24-attestation-registry)'s
exact pattern, applied to SEP-31 instead of SEP-24:

- `sep31-conformance` — the checking library + CLI. Produces the results this contract stores.
- **This repo** — the on-chain record.
- `sep31-conformance-backend` — the API service that runs the checker and writes to this contract.

(Cross-repo links above are left as plain names rather than URLs until
all three repos are pushed under their final org — see each repo's own
README once published for the live links.)

## Table of contents

- [Why this exists](#why-this-exists)
- [This contract vs. sep24-attestation-registry](#this-contract-vs-sep24-attestation-registry)
- [Trust model](#trust-model)
- [Data model](#data-model)
- [Interface reference](#interface-reference)
- [Deployed instances](#deployed-instances)
- [Testing philosophy](#testing-philosophy)
- [Development](#development)
- [Deploying your own instance](#deploying-your-own-instance)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

## Why this exists

`sep31-conformance` can tell you, right now, whether a receiving anchor's
SEP-31 discovery surface matches spec. But that result only exists wherever the check
happened to run. A sending anchor deciding whether to trust a corridor —
or a directory site listing verified corridors — needs a durable,
independently queryable answer instead of a one-off terminal output. This
contract is that: an off-chain backend runs the real check, and only on a
pass, submits a signed attestation here.

## This contract vs. sep24-attestation-registry

The underlying problem — "durably record a pass/fail + hash for a domain,
signed by one admin key" — is identical regardless of which SEP is being
attested to, so this contract's Rust source is structurally the same as
[`sep24-attestation-registry`](https://github.com/SEP-24-conform/sep24-attestation-registry)'s.
That's deliberate reuse of a proven, already-audited-in-full pattern, not
duplicated effort by accident — see that repo's own
[Design decisions](https://github.com/SEP-24-conform/sep24-attestation-registry#design-decisions)
for the reasoning behind every choice here (hash-not-full-report, no
history log, no built-in multi-sig). What differs is only the surrounding
context: a separate deployment, a separate admin key, and attestations
that mean "this SEP-31 corridor conforms" rather than "this SEP-24
anchor conforms."

## Trust model

Identical to
[`sep24-attestation-registry`'s trust model](https://github.com/SEP-24-conform/sep24-attestation-registry#trust-model) —
trust the admin key to only submit attestations reflecting real
conformance runs; every write is a permanent, signed, publicly-visible
transaction; the checker being open source makes the admin's claims
independently falsifiable rather than merely asserted.

## Data model

```rust
pub struct Attestation {
    pub timestamp: u64,       // ledger close time (unix seconds) when written
    pub passed: bool,         // whether the conformance run had zero failures
    pub result_hash: BytesN<32>,  // hash of the full report, e.g. SHA-256
}
```

A single map from domain (`String`) to its most recent `Attestation`,
overwritten on each new `attest` call — current status, not a history log.

## Interface reference

| Function | Auth required | Description |
|---|---|---|
| `initialize(admin: Address)` | — | One-time setup. Panics if already initialized. |
| `get_admin() -> Address` | — | Returns the current admin address. |
| `set_admin(new_admin: Address)` | current admin | Rotates the admin key. |
| `attest(domain: String, passed: bool, result_hash: BytesN<32>)` | admin | Records a conformance result for `domain`, overwriting any prior attestation for the same domain. |
| `get_attestation(domain: String) -> Option<Attestation>` | — | Reads the latest attestation for `domain`. `None` if never attested. |

## Deployed instances

| Network | Contract ID |
|---|---|
| Testnet | [`CBWQIB3544K3DV3TIUX3WBNA65DHDCK35WJUSUTFWAJSDGOYP24SP2HF`](https://stellar.expert/explorer/testnet/contract/CBWQIB3544K3DV3TIUX3WBNA65DHDCK35WJUSUTFWAJSDGOYP24SP2HF) |
| Mainnet | not yet deployed |

Round-trip verified for real on this testnet deployment: an attestation
for `testanchor.stellar.org` was written and read back correctly,
including the real ledger-assigned timestamp — not just exercised through
the unit test suite in isolation.

## Testing philosophy

7 unit tests, identical in shape and reasoning to
[`sep24-attestation-registry`'s test suite](https://github.com/SEP-24-conform/sep24-attestation-registry#testing-philosophy),
including the same negative-auth test
(`attest_fails_without_the_admins_authorization`) that builds a fresh
`Env` without `mock_all_auths()` specifically to prove the admin-only
gate is load-bearing rather than untested-but-assumed-correct.

## Development

```sh
cargo test              # 7 unit tests, including the negative auth test above
stellar contract build   # -> target/wasm32v1-none/release/registry.wasm (~2.9KB)
```

## Deploying your own instance

```sh
stellar keys generate sep31-registry-admin --network testnet --fund
stellar contract build
stellar contract deploy \
  --wasm target/wasm32v1-none/release/registry.wasm \
  --source sep31-registry-admin --network testnet --alias registry
stellar contract invoke --id registry --source sep31-registry-admin --network testnet -- \
  initialize --admin "$(stellar keys address sep31-registry-admin)"
```

As with the sibling project, the deploying key is not necessarily the key
that should sign attestations day to day — rotate admin via `set_admin`
to a dedicated key held only by whatever backend operates this instance.

## FAQ

**Why not just reuse `sep24-attestation-registry`'s deployed contract
instance for SEP-31 attestations too, since the code is identical?**
Because the two record fundamentally different claims — "this domain's
SEP-24 anchor conforms" and "this domain's SEP-31 corridor conforms" are
not the same statement, and a wallet or sending anchor querying this
contract needs to know unambiguously which claim it's reading. Separate
deployments keep that unambiguous; sharing one instance would require
namespacing attestations by SEP within the key, adding complexity to
avoid a code-reuse question that reusing the *source*, not the
*deployment*, already answers more simply.

**Does this contract know or care what SEP-31 actually requires?** No —
same as its sibling, it has no opinion on what "conformant" means; that
logic lives entirely in `sep31-conformance`. This contract only stores
whatever it's told, gated by the admin key.

See [`sep24-attestation-registry`'s own FAQ](https://github.com/SEP-24-conform/sep24-attestation-registry#faq)
for the rest (admin key loss, who can read, why a plain boolean) — all of
it applies here unchanged.

## Contributing

Same extension points as the sibling repo (attestation history, richer
result data, admin governance) — see its
[Contributing](https://github.com/SEP-24-conform/sep24-attestation-registry#contributing)
section. Given the source is intentionally near-identical, a proposal
that makes sense for one likely makes sense for both; consider linking
issues across both repos rather than solving the same design question
twice independently.

## License

Apache-2.0
