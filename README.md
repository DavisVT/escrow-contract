# Milestone Escrow Contract

A Soroban smart contract on Stellar that enables trustless milestone-based escrow between a client, freelancer, and arbiter.

## Overview

This contract allows a client to fund a job broken into milestones. Funds are locked in the contract and released per milestone only when the client approves delivery. Disputes can be raised by either party and resolved by a designated arbiter.

## Trust Model

The escrow's protection depends on **four distinct roles** assigned at `initialize` time:

- **Admin** — the platform operator. Holds admin-only endpoints (whitelist management, pause/resume, admin overrides, upgrades). The admin is expected to be a neutral platform address, not the client or freelancer.
- **Client** — funds the job and approves milestone releases.
- **Freelancer** — delivers milestones and receives payouts.
- **Arbiter** — a neutral third party that resolves disputes via `resolve_dispute` and `apply_dispute_arbitration_split`.

`initialize` enforces party distinctness: it rejects any collision among `admin`, `client`, `freelancer`, and `arbiter` with `Error::InvalidParties`. This prevents the client from resolving their own disputes (arbiter == client), the freelancer from ruling in their own favour (arbiter == freelancer), or the client from holding every admin override (admin == client). The frontend must pass four genuinely distinct addresses — it must not reuse the connected wallet as both admin and client.

## Contract Functions

| Function | Caller | Description |
|---|---|---|
| `initialize` | Anyone | Set up job with milestone amounts and spec hashes, a 30-day dispute timeout and Upfront funding |
| `initialize_with_dispute_timeout` | Anyone | Set up job with a custom, non-zero dispute timeout |
| `initialize_with_funding_mode` | Anyone | Set up job with Upfront or PerMilestone funding |
| `fund` | Client | Deposit total amount into contract (Upfront mode) |
| `fund_milestone` | Client | Deposit one milestone (PerMilestone mode) |
| `mark_delivered` | Freelancer | Mark a milestone as delivered |
| `request_revision` | Client | Return a delivered milestone to Pending (max 3) |
| `approve_milestone` | Client | Release funds for a delivered milestone |
| `raise_dispute` | Client or Freelancer | Freeze a milestone for arbitration |
| `escalate_dispute` | Client or Freelancer | After the timeout, restore a disputed milestone to its previous status |
| `resolve_dispute` | Arbiter | Release to freelancer or refund to client |
| `get_job` | Anyone | View current job state |
| `sweep_excess` | Client or admin | Recover surplus tokens not backing obligations |
| `emergency_unpause_parties` | Client and freelancer | Jointly lift a party-initiated pause |

## Contract Errors

| Error | Description |
|---|---|
| `AlreadyInitialized` | Contract has already been initialized |
| `AlreadyFunded` | Job has already been funded |
| `AlreadyAccepted` | Job has already been accepted by the freelancer or arbiter |
| `InvalidAmount` | Invalid monetary amount (zero, negative, or overflow) |
| `InvalidDuration` | Invalid time duration (e.g., `auto_release_seconds` is zero) |
| `InvalidDisputeTimeout` | Dispute timeout is zero or invalid |
| `InvalidParties` | Party addresses are not distinct (collision detected) |
| `InvalidAddress` | Address is the zero account or zero contract |
| `InvalidStatus` | Operation not allowed in current milestone/escrow state |
| `InvalidMilestone` | Milestone index out of bounds or milestone data invalid |
| `InvalidRatio` | Invalid multisig ratio (empty, negative, or sum overflow) |
| `TooManyParties` | Multisig ratio vector exceeds maximum length (255) |
| `Unauthorized` | Caller is not authorized for this operation |
| `NotInitialized` | Contract has not been initialized |
| `NotFunded` | Job has not been funded yet |
| `NotAccepted` | Freelancer or arbiter has not accepted the job |
| `Paused` | Contract is emergency-paused |
| `TokenNotWhitelisted` | Token is not in the contract's whitelist |
| `TokenAlreadyWhitelisted` | Token is already in the whitelist |
| `WhitelistFull` | Token whitelist is at maximum capacity (50) |
| `WhitelistMinimum` | Cannot remove last token from whitelist (minimum 1 required) |
| `EmptyBalance` | Contract has no token balance for the operation |
| `ArithmeticOverflow` | Arithmetic operation would overflow |
| `RevisionLimitExceeded` | Maximum revision requests (3) exceeded for a milestone |
| `DisputeTimeoutLocked` | Dispute timeout configuration is locked |
| `TaxWithholdingLocked` | Tax withholding configuration is locked |
| `PlatformFeeAllocationLocked` | Platform fee allocation is locked |
| `EmergencyPauseLocked` | Emergency pause configuration is locked |
| `TimeExtensionLocked` | Time extension configuration is locked |
| `PaymentStreamingLocked` | Payment streaming configuration is locked |
| `NoPendingAdminTransfer` | No pending admin transfer to accept or cancel |
| `AdminTransferNotExpired` | Admin transfer timeout has not expired |
| `UninitializedColdConfig` | Cold configuration (whitelist, multisig, yield, fees) not initialized |
| `AlreadyMigrated` | Cold configuration has already been migrated |

## Milestone States

Pending → Delivered → Released

↓

Disputed → Released (arbiter favors freelancer)

→ Refunded (arbiter favors client)

The full state machine, including every entry point, its required signers and the escrow-level locks, is specified in [`contracts/milestone-escrow/SPEC.md`](contracts/milestone-escrow/SPEC.md).

## Security

Roles, trust assumptions, admin powers and known risks are described in [`THREAT_MODEL.md`](THREAT_MODEL.md).

## Prerequisites

- https://rustup.rs/ (the toolchain is pinned to 1.96.0 by `rust-toolchain.toml`)
- https://developers.stellar.org/docs/smart-contracts/getting-started/setup (stellar-cli 28.0.0)
- wasm32 target: `rustup target add wasm32v1-none`
- Docker, for reproducible builds

## Build

For development, from the repository root:

```bash
stellar contract build --manifest-path contracts/milestone-escrow/Cargo.toml
```

Anything that will be deployed must come from the reproducible build instead:

```bash
scripts/build-reproducible.sh
```

It compiles a clean export of `HEAD` (or of the commit given as the first
argument) inside a pinned container (`build/reproducible.Dockerfile`: Rust
1.96.0 by image digest, stellar-cli 28.0.0 by release checksum, linux/amd64)
and writes `dist/milestone_escrow.wasm`, its SHA-256 and `dist/build-info.json`.
The same commit gives the same hash on any machine. CI runs the same script on
every commit and publishes the hash in the job summary and the wasm as an
artifact.

## Test

From the repository root:

```bash
cargo test -p milestone-escrow
```

## Project Status

- Mainnet release gates are tracked in the [mainnet readiness checklist](MAINNET.md)
- The admin authorization direction and migration decision are recorded in [ADMIN_MULTISIG_DECISION.md](ADMIN_MULTISIG_DECISION.md)
- Contract implementation completed in `contracts/milestone-escrow`
- Includes milestone delivery, approval, dispute, and arbitration flows
- Initialize now rejects empty milestone lists, zero/negative amounts, zero addresses, zero auto-release windows, and **party collisions** (`admin == client`, `admin == freelancer`, `arbiter == client`, `arbiter == freelancer`, `client == freelancer`) with explicit contract errors
- `raise_dispute` now performs zero-address check, `require_auth`, and identity validation **before** writing the re-entrancy lock (issue #566), matching the authorization pattern used by `fund`, `approve_milestone`, and `resolve_dispute`; includes the five precondition guard assertions present in all other mutating functions
- Disputes may only be raised on delivered or partially released milestones. The default dispute timeout is 30 days; `initialize_with_dispute_timeout` allows a custom non-zero timeout. If neither party gets an arbitration decision by that deadline, either party may call `escalate_dispute` to return the milestone to its pre-dispute status; a restored delivered milestone gets a fresh auto-release window
- `lock_escrow_interest_yield` now rejects unauthorized callers with `Unauthorized` and an illegal source state with `InvalidStatus` **before** it reads or writes the ledger entry (issue #464); authorization runs first via `require_admin_from_instance`, so a non-admin cannot probe whether an escrow is locked, and an already-locked configuration is refused rather than silently re-locked
- Contract tests and snapshots are provided under `contracts/milestone-escrow/test_snapshots`

## Deploy (Testnet)

```bash
scripts/build-reproducible.sh
stellar contract deploy \
  --wasm dist/milestone_escrow.wasm \
  --network testnet \
  --source <your-account>
```

Then record the contract ID, `source_commit` and `wasm_sha256` from
`dist/build-info.json` in `deployments/<network>.json`, and the hash in
`CHANGELOG.md`.

After upgrading an existing deployment to this version, the admin should call
`migrate_cold_config` once to move the whitelist, multisig, yield and fee
configuration out of instance storage (issue #772). The contract keeps working
before that call; the migration only makes every later call cheaper.

## Verifying the deployed contract

Anyone can check which source a deployed contract was built from:

1. Read the installed wasm hash:

   ```bash
   stellar contract fetch --id <contract-id> --network testnet --out-file deployed.wasm
   sha256sum deployed.wasm
   ```

   (`stellar contract info` and the explorer show the same hash.)
2. Look up `source_commit` for that contract in `deployments/<network>.json`.
3. Rebuild that commit and compare:

   ```bash
   scripts/build-reproducible.sh <source_commit>
   cat dist/milestone_escrow.wasm.sha256
   ```

The two hashes must be identical. If they differ, the deployed code is not the
reviewed source at that commit.

## License

MIT

## Deployed Contract

| Network | Contract ID |
|---|---|
| Testnet | `CDD5WKK3WT3QVKXMXTJNDIXE4T73FK6GGXDSD6UTJAH6YYZU52SQ4MUH` |

Deployment records, including the installed wasm hash, are in
[`deployments/`](deployments/). The current testnet deployment predates
reproducible builds, so its source commit is unknown.

Explorer: `https://stellar.expert/explorer/testnet/contract/CDD5WKK3WT3QVKXMXTJNDIXE4T73FK6GGXDSD6UTJAH6YYZU52SQ4MUH`
