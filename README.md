# EQTY Faucet

A headless CPU proof-of-work faucet for people and AI agents, promoting EQTY.

**Status: specification only. The service, CLI and website are not released.**

The first release will distribute small amounts of **testnet EQTY ERC-20 tokens on Base Sepolia**. Native testnet ETH is a later extension. Test EQTY cannot pay gas fees.

Planned components:

- Rust service with persistent credit accounting and recoverable ERC-20 payouts.
- Rust client library and `eqty-faucet` CLI with the public endpoint built in.
- Agent skill for bounded headless use.
- Explanation, installation and status page at `https://faucet.eqty.me`.

Read the [specification](spec.md) and [delivery tickets](docs/tickets.md).

Proof-of-work is an abuse-control mechanism only. There is no revenue mining, captcha, social login or recipient private-key requirement.
