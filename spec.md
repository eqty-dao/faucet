# EQTY Faucet specification

Date: 2026-11-25
Repository: https://github.com/eqty-dao/faucet
Public entry point: https://faucet.eqty.me
Status: specified, not implemented or deployed

## 1. Purpose and product boundaries

Promote EQTY by providing an agent-friendly, fully headless faucet. Users run a CLI that performs CPU-oriented proof-of-work locally and gradually earns very small amounts of test tokens. Obtaining tokens requires no browser, captcha, social login, existing cryptocurrency, or private key for the recipient wallet.

The first release distributes **testnet EQTY ERC-20 tokens only**. Use Base Sepolia as the initial network. Native testnet currencies such as Ethereum Sepolia ETH and Base Sepolia ETH are a later extension, not part of this release. Test EQTY is not gas and cannot pay network transaction fees. The service funds its own native gas for outgoing ERC-20 transfers; recipients do not need gas to receive tokens but need it to use them on-chain.

Provide four deliverables in this repository:

1. A Rust HTTP service with persistent accounting and ERC-20 payouts.
2. A reusable Rust client library and a thin `eqty-faucet` CLI.
3. An agent skill describing installation, bounded execution, recovery, and payout verification.
4. A public explanation and status page at `faucet.eqty.me`.

The site promotes EQTY and explains the service. It has no browser miner, wallet connection, account registration, or claim form. Agents and people obtain tokens through the same CLI/library protocol. A separate MCP server, TypeScript/Python SDKs, mobile clients, and automatic background startup are out of scope.

## 2. Distribution and economics

Clients continuously search for small valid proofs. Each accepted proof earns a tiny, fixed amount of off-chain token credit. Running longer produces more credit through additional proofs, rather than selecting a large reward and receiving one correspondingly large puzzle.

Proof-of-work serves only as abuse resistance. Do not mine Monero, sell computation, create tradable credit tokens, pay rewards with monetary value, or finance hosting by exchanging user work. Hosting and treasury funding are operator responsibilities. Credit is an internal entitlement to this faucet's test tokens, not an on-chain asset.

Set the work-to-reward ratio deliberately low: convenient for an agent needing some test tokens, unattractive for bulk collection. This is a policy objective, not a guarantee against free compute, botnets, or large CPU fleets. Test EQTY has no established market price; do not claim proven profitability thresholds or compare its token quantity directly with ETH faucet quantities.

Keep these operator parameters explicit, versioned, and advertised before work begins:

- Reward in integer token base units per accepted proof.
- PoW algorithm/version, parameters, and required difficulty.
- Minimum payout, maximum payout, and optional automatic payout threshold.
- Global credit issuance budgets per hour/day, pending-liability ceiling, and payout/gas budgets.
- Session/challenge lifetime and admission/concurrency limits.

Determine initial numeric values through benchmarks and a treasury budget before public launch. Do not invent token amounts or change the reward/difficulty of an already issued challenge. Changes apply to new challenges. Reserve enough issuance budget for the bounded number of proofs authorized by outstanding challenges; expiry releases unused reservations. Advertise exhausted capacity before asking clients to compute.

## 3. User and agent flow

Example intended interface, not a presently installed command:

```sh
eqty-faucet mine --network base-sepolia --address 0x... --threads 1 --duration 5m
eqty-faucet mine --network base-sepolia --address 0x... --threads 1 --until 10 --json
eqty-faucet status --json
eqty-faucet payout --json
```

`--until` means an explicitly labelled amount of test EQTY, not ETH or native gas. Parse human amounts without floating-point rounding. Document whether the target is total earned in this run or the session total; use **total earned in this run** in v1 so previous payouts do not make the target ambiguous.

The CLI has the public API base `https://faucet.eqty.me/api/v1` built in. No endpoint lookup or configuration is needed for ordinary use. Allow an explicit endpoint override for local development/self-hosting; never silently redirect a production session or send its bearer credential to another origin.

1. Show the selected chain ID, asset contract/symbol/decimals, reward policy, payout threshold, and CPU limits.
2. Create or resume a mining session bound immutably to the recipient, chain, and asset.
3. Obtain a challenge, solve continuously with the chosen worker count, submit proofs, and display accepted credit.
4. Renew challenges as needed. At the automatic threshold, request payout while mining continues within the requested limits.
5. Stop on the requested duration/target, Ctrl-C, or an actionable error. Preserve accepted credit and resume metadata. A reached threshold may trigger a final payout; below-threshold credit stays available for later runs.
6. Track payout status to a confirmed transaction and return a transaction hash and explorer link. A timed-out or pending payout must never be reported as received.

One thread is the default. Mining is explicit and foreground-only; no hidden daemon. Duration/target flags support unattended runs, and the skill must always choose an explicit bound. JSON mode emits documented machine-readable events to stdout and diagnostics to stderr. Include stable exit codes for success, invalid input, policy pause, recoverable network failure, and permanent protocol failure. A clean interruption preserves progress and does not pretend an unfinished payout is complete.

## 4. Rust architecture

Use a Rust workspace with clear responsibility boundaries:

```text
crates/protocol/   Versioned wire types, canonical challenge encoding, amount types
crates/pow/        Solver/verifier wrapper around an existing CPU-oriented algorithm
crates/service/    HTTP API, durable ledger, budgets, sessions, payout worker
crates/client/     Async API client, solver orchestration, resume and payout tracking
crates/cli/        User commands, limits, progress, JSON and exit codes
skills/eqty-faucet/  Agent skill and supporting instructions
site/             Public explanation, installation and service status
```

Use PostgreSQL for persistent sessions, policy/challenge versions, proof acceptance, double-entry or equivalently auditable credit accounting, budget reservations, and payout jobs. Database transactions and unique constraints enforce invariants across concurrent requests and process restarts. The client library owns the complete flow; the CLI must not duplicate protocol or retry logic. Pin the Rust toolchain and PoW implementation/dependencies.

## 5. Proof-of-work and abuse resistance

HashWX is the preferred candidate because the desired workload favors general-purpose CPUs and requires cheap server verification. This is a **candidate to validate**, not a verified deployment choice. Before freezing protocol v1, benchmark maintained x86-64 Linux, ARM64 macOS and x86-64 Windows MSVC builds, CPU/GPU advantage using available evidence, verification throughput, memory, native dependency integration, and license compatibility. Record the decision and reproducible vectors. If unsuitable, record a replacement decision before implementation proceeds; never roll a custom cryptographic primitive.

Do not use reported client speed to lower difficulty. Clients may parallelize, but faster hardware must perform the same required work per credit. CPU-centric does not mean equal performance per machine. Do not reuse RandomX/Monero jobs or replace the algorithm with a GPU-friendly static hash without an explicit protocol decision.

Each server-authenticated challenge binds a domain separator, protocol and algorithm versions, session ID, random challenge ID/seed, chain ID, token address, recipient, reward, difficulty, parameters, proof-count limit, and expiry. The solver adds a nonce. Define byte-exact canonical encoding, integer widths, and accepted nonce space with test vectors. Use short-lived challenges that allow a bounded set of distinct proofs, avoiding a request for each failed hash. Credit each `(challenge_id, nonce, proof)` at most once; challenge fields cannot be altered to increase rewards or redirect credit.

Authenticate sessions with a high-entropy opaque resume credential. Store only its hash server-side and persist it locally with restricted permissions. Bind every balance, proof, and payout operation to its session. A credential cannot change the recipient or asset; wallet addresses alone are not authentication. The receiving wallet's private key is never requested. Multiple sessions/wallets do not create global issuance capacity.

Combine PoW with bounded HTTP bodies, challenge admission, connection limits, bounded verifier queues, per-source heuristics, global budgets, and overload backoff. Check authentication, size, encoding, expiry, replay, and available capacity before expensive verification. Never trust a client-provided hash without recomputing it. Cheap fake hashes can still force verification, so queue limits remain necessary. Treat shared cloud IPs as a signal, not evidence that an agent is illegitimate. Do not rely on per-IP or per-wallet quotas to solve Sybil attacks.

## 6. HTTP contract

Serve JSON over HTTPS under `/api/v1`, with an OpenAPI document, protocol version, bounded request sizes, structured errors, and documented retry semantics. Proposed resources:

| Endpoint | Responsibility |
| --- | --- |
| `GET /assets` | Enabled testnet assets and advertised policy, availability and explorer data |
| `POST /sessions` | Create recipient-bound session and return its resume credential once |
| `POST /sessions/{id}/challenges` | Issue an authenticated, budget-reserved PoW challenge |
| `POST /sessions/{id}/proofs` | Verify and credit a proof atomically; duplicate retry returns original result |
| `GET /sessions/{id}` | Authenticated credited, reserved, paid and available balances and current jobs |
| `POST /sessions/{id}/payouts` | Reserve eligible credit and create an idempotent payout job |
| `GET /sessions/{id}/payouts/{payout_id}` | Return durable payout state, transaction hashes and confirmations |
| `GET /status` | Public availability, policy pause and supported assets, without session/private data |
| `GET /health/live`, `GET /health/ready` | Process liveness and dependency readiness |

Require session authentication after creation. Accept an idempotency key for payout creation; store its request digest and reject reuse with different input. Define error codes including invalid proof, stale challenge, duplicate proof, unauthorized session, insufficient credit, capacity exhausted, treasury unavailable, overloaded, unsupported asset/version, and payout pending. Return retry hints for temporary pauses. Client retries must query/reconcile accepted proof and payout state before creating new work.

## 7. Test EQTY treasury and payouts

V1 enables one testnet EQTY ERC-20 asset on Base Sepolia. The existing EQTY contracts README lists chain ID `84532` and token `0x24159513a74ca294f5367764557438d318eb7ffe` as a starting reference. This address is **not live-verified by this specification**. Before launch verify the intended deployment, bytecode, symbol, decimals, transfer behavior, and recipient receipt on the actual network; record the selected address and verification evidence. Confirm funding with the token owner. Do not silently use mainnet, deploy a substitute token, or grant the faucet token-minting authority.

The service distributes an operator-funded ERC-20 inventory using ordinary transfers. Keep its signer secret out of source, client state, logs, page content, and API responses. Fund its native gas separately. Admission must account for spendable token inventory, outstanding accepted credit, budget-reserved challenges, reserved payouts, and a gas safety margin; do not issue unbacked credit. Check the RPC chain ID against the configured testnet allowlist at startup and before signing.

Maintain a payout state machine: `queued -> prepared -> broadcast -> confirmed`, with explicit `failed` and `reconciling` paths. Persist nonce, signed transaction/hash and reserved credit before broadcasting. An uncertain RPC response or restart rebroadcasts/reconciles the same transaction rather than creating a second payment. Track replacements by nonce and receipt, handle dropped transactions/reorgs, and require configured confirmations plus successful receipt and expected ERC-20 Transfer evidence before settling credit as paid. Release reserved credit only after definitive failure; never on an HTTP timeout alone.

Serialize signer nonce allocation and enforce uniqueness even with multiple workers. Bound gas, attempts and pending jobs. Low inventory/gas or RPC failure pauses new work cleanly, preserves existing credit and exposes a truthful status. Provide a stop-issuance/payout operational procedure and reconciliation report.

## 8. Agent skill and public page

Publish `skills/eqty-faucet/SKILL.md` and make it discoverable from `https://faucet.eqty.me/skill.md`. The skill must teach installation from versioned releases, checksums, mining with explicit recipient/network/CPU/time limits, resume, JSON parsing, and payout confirmation. It must explain that only test EQTY is available and does not provide gas. No recipient seed/private key, captcha bypass, unrelated wallets, autonomous endless mining, or automatic privilege escalation is required. Agents should stop and report unmet requirements instead of switching funds or networks.

The page includes:

- EQTY branding, project purpose and links to EQTY and the source repository.
- Exact current asset/network/contract, work/reward policy and payout minimum.
- Copyable installation and mining examples for Linux x86-64/ARM64, macOS ARM64/x86-64 and Windows x86-64, with the verified supported matrix stated explicitly.
- Library entry point, CLI help, agent skill and machine-readable API links.
- Current operational status and clear maintenance/treasury pause messages.
- A plain explanation of CPU use, saved credit, recipient privacy and retained service data.
- A clear test-token label and a statement that recipient transactions still need native gas.

No web claim flow or browser mining. Serve the page and API at the agreed domain with TLS; site content can be static. Do not force a particular hosting vendor or introduce Vercel dependencies. Configure DNS and production secrets during the deployment ticket, not while writing this spec.

### Client builds and distribution

[EQFAUCET-7](http://localhost:3456/tasks/107) owns client distribution, separate from CLI behavior and public service deployment. Keep the service, client, native PoW integration, release workflows, Homebrew formulae and Scoop manifests in this monorepo.

- Build and test the client on native Linux, macOS and Windows GitHub Actions runners. Windows starts with `x86_64-pc-windows-msvc`, using a Windows runner such as `windows-2025`, including the native PoW build, solver/verifier compatibility tests and CLI tests. Windows ARM64 is deferred.
- Publish versioned client archives, checksums, required license files and build provenance through GitHub Releases. Verify the packaged artifacts and tie each channel to the same tested source revision.
- Use an EQTY Homebrew tap for macOS, a Launchpad PPA for Ubuntu, and an EQTY Scoop bucket for Windows. Keep formulae/manifests in the monorepo and update artifact URLs/checksums from the release. Launchpad builds Ubuntu binary packages from signed source uploads.
- Publish the reusable Rust client library and required public dependency crates on crates.io. Document source-build prerequisites for Cargo users.
- The site and skill teach package-manager installation and upgrades. Do not distribute or recommend a downloaded shell installation script or curl-to-shell. WinGet and Chocolatey are not the selected Windows channels.
- Only the service needs a Linux deployment build. Public homebrew/core or Ubuntu Universe inclusion is a later option, not a launch requirement.

Launchpad ownership, signing credentials, Ubuntu series, package/crate name availability, release-channel authentication and any manual publication steps remain to be resolved in the distribution story. No release, PPA, tap, bucket or crate is claimed to exist yet.

## 9. Verification and release gates

The implementation must demonstrate:

1. Solver/verifier compatibility and tamper rejection using reproducible vectors on the supported CPU architectures.
2. Exactly-once credit for duplicate/concurrent submissions, cross-session replay rejection, expiry and immutable quote enforcement, and budget exhaustion with outstanding reservations.
3. Persistent credit/resume after client/service restart, with authenticated session access and no credential leaks.
4. Exactly-once payout intent/reservation under concurrent requests, restart recovery after signing/broadcast, nonce uniqueness, RPC timeouts, failed receipts, and reorg reconciliation.
5. Real test-EQTY transfer to an authorized test recipient on Base Sepolia, with receipt, Transfer log and balance delta. No native token distribution is included.
6. End-to-end CLI and library flow, bounded CPU/time, interruption/resume, target semantics, JSON/exit codes, overload backoff and truthful pending status.
7. Published client artifacts and package metadata from the distribution story, verified Homebrew/PPA/Scoop installation, checksums, a Linux service container/deployment artifact, database migration/backup/restore procedures, and green CI for the exact released revision.
8. TLS page/API at `faucet.eqty.me`, readable skill/API docs, working links, and site/CLI values matching the deployed asset policy.

Security-sensitive tests must exercise concurrency and uncertain outcomes, not only mocked happy paths. Use local EVM fixtures for repeatability and one recorded live acceptance run for actual network behavior. No mainnet test transactions or unrelated wallet funds.

## 10. Delivery breakdown

Use seven substantial tickets on the EQTY Faucet board, including a separate client distribution story. Each implementation ticket includes relevant tests and documentation; do not split every endpoint into its own ticket.

| Key | Ticket | Dependencies |
| --- | --- | --- |
| F1 | Establish the Rust workspace, protocol and CPU PoW baseline | None |
| F2 | Build the persistent faucet API, credit ledger and abuse controls | F1 |
| F3 | Implement funded test-EQTY ERC-20 payouts and recovery | F2 |
| F4 | Deliver the Rust client library and headless CLI | F1; final integration with F2/F3 in F6 |
| D1 | EQFAUCET-7: install the client through native package managers | F4 |
| F5 | Publish the agent skill and EQTY explanation page | F4, D1 |
| F6 | Deploy, release and verify the complete test-EQTY faucet | F3, F4, F5, D1 |

F1 records exact reward/work parameters or a measured calibration policy before public launch. F3 verifies deployment identity and treasury/gas funding. F4 owns cross-platform library/CLI behavior; D1 owns client release packaging and publication. F5 documents those installation routes. F6 selects the hosting target, applies DNS/secrets and verifies installation and end-to-end operation. These are explicit delivery gates, not assertions that resources are already provisioned.

## 11. Later extension

Keep asset identity separate from PoW, ledger and session identity: chain ID, asset kind, contract address if applicable, decimals, budgets and transfer adapter. V1 accepts only its configured test EQTY asset. A later change may enable native testnet ETH through a separate payout adapter, funding policy, benchmarks and tickets. Mainnet token distribution, cross-chain bridges, minting and third-party funding grants remain outside v1.

## References

- [HashWX candidate algorithm](https://github.com/tevador/hashwx)
- [Existing PoWFaucet](https://github.com/pk910/PoWFaucet)
- [EQTY contract deployment reference](https://github.com/eqty-dao/eqty-contracts#deployed-addresses)
- [Sui headless PoW flow reference](https://faucet.sui.io/how-to-use/)

These are design references, not a claim that their code, funding or deployments are part of this repository.
