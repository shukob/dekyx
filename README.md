# DeKYX — Decentralized Know Your X

DeKYX is the reusable identity and participation-qualification layer. It is not
an Aethel subcomponent. `X` may be a person, legal entity, device, service, or
autonomous agent.

## Boundary

- DeKYX owns issuer trust and key epochs, KYC/KYB subject type, qualification
  attributes, selective disclosure, anonymous zero-knowledge presentation,
  revocation, key rotation, exact scope/request binding, and presentation
  replay consumption.
- DeCCP consumes a verified qualification when admitting a clearing member.
- Aethel consumes a verified qualification before recording a private credit
  decision or guarantee reference.
- zkPI carries an executable instruction; DeFMI updates the authoritative
  ledger. Neither belongs here.

`dekyx-core` has no application dependency. `dekyx-aethel` is an explicit
adapter that shapes Aethel's request and returns the subject-binding record
Aethel stores beside a credit decision or guarantee. It does not depend on
`aethel-core`; `aethel-core` depends on it.

This directory is a standalone, locked Rust workspace rather than a crate added
to QOMM. The enclosing TradFi repository ignores new `mvp/*` projects, matching
the intended later publication as its own repository. No nested Git history or
remote is created here.

## Objects and their owners

| Object | Produced by | Verified by |
|---|---|---|
| `IssuerDefinition` (issuer id, key epoch, key, namespace, subject kinds, window) | the deployment's governance | `IssuerDirectory` on registration and on every reload |
| `Credential` (signed scope commitment, policy, qualification root) | `CredentialIssuer`, after a holder issuance proof | `DeKyxVerifier` |
| `RevocationStatusList` (signed, epoch-ordered) | the issuer key of the epoch it covers | `IssuerDirectory::publish_status_list`, then every presentation |
| `AnonymousPresentation` | the holder, for one exact `PresentationContext` | `DeKyxVerifier::verify_eligibility` |
| `VerifiedEligibility` / `AethelSubjectBinding` | DeKYX only | stored by the application |

## Privacy, lines, and unlinkability

The issuer signs a scope-specific Pedersen commitment, a policy digest, and a
Merkle root of qualifications. The holder discloses only the required
qualification leaves and proves in zero knowledge that the signed commitment
and scope nullifier contain the same secret. The proof challenge includes the
audience, action, request, nonce, and expiration context, so a transcript made
for one artifact cannot be moved to another.

The holder, not the issuer, generates the subject secret. Issuance requires a
zero-knowledge proof that the holder knows the opening of the requested
commitment, so the normal API never passes the secret or blinding scalar to the
issuer. Every eligibility requirement names the exact issuer id, key epoch, and
issuer namespace digest; merely being present in a broad registry is not enough
to satisfy an unrelated policy.

A **subject line** is `H(issuer, scope, policy, scope nullifier)`. It excludes
the issuer key epoch and the commitment blinding, so a credential re-issued
under a rotated issuer key, or with a fresh blinding, continues the same line
as long as the same subject secret is presented in the same scope. Two scopes
use unrelated nullifier bases and re-randomized commitments
(`CredentialWitness::rerandomize`), so nothing a verifier sees links them.

This first contract is scope-pseudonymous, not a claim of issuer-unlinkable
anonymous credentials: the Ed25519-signed commitment is stable inside one
credential and can be correlated with the issuance record. A later BBS+/CL or
equivalent rerandomizable-signature adapter can strengthen that property
without changing the provider boundary.

## Revocation, key rotation, and replay

Revocation lists are issuer-signed, epoch-ordered, canonical sets of credential
digests. A stale list fails closed, an older list cannot replace a newer one,
and an epoch without a published list cannot verify anything
(`DeKyxError::MissingStatusList`).

`IssuerDirectory` holds every accepted key epoch of every issuer and the latest
status list per epoch. `rotate_key` registers a strictly higher epoch with a
different key and bounds the earlier epochs to a grace instant; a grace instant
before their `valid_from` revokes them immediately, which is the key-compromise
path. The directory serializes as plain data and deserializes only through the
same validation that live registration performs.

`PresentationLedger` consumes the (scope nullifier, context) pair so that a
re-randomized proof of the same credential for the same context is a replay.
The ledger is monotone; an application that persists it must authenticate the
persisted copy, because a truncated ledger re-enables replay.

## Persistence boundary

- `IssuerRegistry` and `IssuerDirectory` cannot be deserialized around their
  validation (`IssuerRegistry` has no `Deserialize`; `IssuerDirectory` uses a
  validated `TryFrom` record).
- Credentials, status lists, presentations, and contexts are wire types.
- `VerifiedEligibility` is deliberately not serializable: it is the output of a
  verification, not an input.

## Aethel integration

`aethel-core` (in `mvp/qomm/rust`) depends on `dekyx-core` and `dekyx-aethel`
by sibling path. Aethel keeps the DeKYX `IssuerDirectory` inside its book,
records which credential-issuer provider vouches for each DeKYX issuer key,
publishes issuer-signed status lists, and hands every holder presentation to
`AethelDeKyxAdapter::verify` for the exact credit decision or guarantee it is
bound to. Aethel stores only the returned `AethelSubjectBinding`. The former
Aethel-owned credential, proof, and issuer-capability verification code is
gone, and the legacy `qomm-zkpi::confidential_subject` module has been
deleted: DeKYX is the only implementation of KYB and anonymous presentation.

The Avalanche VM in `mvp/qomm` uses DeKYX a second time, through DeCCP's
`EligibilityPort`: an Aethel guarantor joins the DeCCP clearing book by
presenting a credential for the clearing-membership scope, and DeCCP records
only the resulting subject line.

## Publication

`mvp/qomm/rust/qomm-harness/src/bin/export_repos.rs` publishes this
workspace as the `dekyx` repository (manifest, lock, `crates/`, this README,
and the shared MIT `LICENSE`; the workspace manifest declares MIT to match).
`aethel` and `defmi` take it as a Git dependency, so it is exported and pushed
before them.

## Verification

All tests, Clippy, formatting, and builds run on an approved remote Linux
worker; the local Mac is for reading, editing, and `cargo fmt`. The latest run
(OmenX, Rust 1.97.1 and the 1.85.1 MSRV with `--locked`): 6 core and 2 adapter
integration tests passed, warning-denied Clippy passed, formatting passed,
optimized release build passed.
