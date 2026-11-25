# Delivery tickets

Vikunja project: [EQTY Faucet](http://localhost:3456/projects/31). Parent project: EQTY.

Scope and acceptance criteria are defined in the [specification](../spec.md). Each ticket contains its implementation scope, boundaries and verification requirements. Nothing is implemented by this documentation commit.

| Ticket | Scope | Blocked by |
| --- | --- | --- |
| [EQFAUCET-1](http://localhost:3456/tasks/93) | Establish the Rust workspace, protocol and CPU PoW baseline | None |
| [EQFAUCET-2](http://localhost:3456/tasks/94) | Build the persistent faucet API, credit ledger and abuse controls | EQFAUCET-1 |
| [EQFAUCET-3](http://localhost:3456/tasks/95) | Implement funded test-EQTY ERC-20 payouts and recovery | EQFAUCET-2 |
| [EQFAUCET-4](http://localhost:3456/tasks/96) | Deliver the Rust client library and headless CLI | EQFAUCET-1 |
| [EQFAUCET-7](http://localhost:3456/tasks/107) | Install the EQTY faucet client through native package managers | EQFAUCET-4 |
| [EQFAUCET-5](http://localhost:3456/tasks/97) | Publish the agent skill and EQTY explanation page | EQFAUCET-4, EQFAUCET-7 |
| [EQFAUCET-6](http://localhost:3456/tasks/98) | Deploy, release and verify the complete test-EQTY faucet | EQFAUCET-3, EQFAUCET-4, EQFAUCET-5, EQFAUCET-7 |

F4 can be developed against protocol fixtures while F2 and F3 are built. F6 performs the final integration, public deployment and live token-transfer acceptance.

EQFAUCET-7 is on the EQTY Faucet board alongside the other faucet tickets. It owns native client builds/tests (including Windows MSVC and PoW), GitHub release packaging, Homebrew/PPA/Scoop distribution and crates.io publication. F4 retains library/CLI functionality; F5 documents installation; F6 verifies the published installation paths and full live flow. No downloaded installation shell script is part of distribution.

V1 pays test EQTY ERC-20 tokens only. Native network test tokens are a separate future scope.
