# agent-exposure

**The most an AI agent can take from your wallet, in one number per token.**

An agent you've given wallet access to may hold several permissions at once: an ERC-20 approval, a Permit2 allowance, a Coinbase spend permission, a Safe allowance, an EIP-7702 delegation, a MetaMask delegation. Each wallet shows its own permission in its own words. agent-exposure adds them up and states the worst case:

> This lets `0xAgent…` take up to **2 USDC in any 24 hours** and **42 USDC in total**, until **31 Dec 2026 23:59 UTC**, to any address. ⚠ "1 USDC a day" is up to 2 USDC within seconds, across a period boundary.

It is **read-only**: no keys, no signing, no transactions, no backend, no data collection. Anything it can't fully read is reported as **unbounded**, never as zero.

## Status

**Specification only (M0, 25 Nov 2026).** Nothing is implemented yet. Read [SPEC.md](SPEC.md): the model, the worst-case rule for each permission type (pinned to the contracts' source), the output format and the test fixtures.

| Milestone | What | When |
| --- | --- | --- |
| M0 | Spec and gap check | Nov 2026 (this commit) |
| M1 | TypeScript library and CLI: ERC-20, Permit2, Coinbase spend permissions, Safe allowances, EIP-7702 on Ethereum and Base | Dec 2026 |
| M2 | Check before granting: MetaMask (ERC-7710/7715) delegations and Coinbase permissions, pasted in | Dec 2026 |
| M3 | MCP server, review, v1 release | Jan 2027 |

Revoking stays in your own wallet. For approvals, [Revoke.cash](https://revoke.cash) does it well, and agent-exposure links there.

## Security

agent-exposure is a security tool, so a wrong "bounded" figure is the bug that matters most. Please report one privately through this repo's **Security → Report a vulnerability**. See [SECURITY.md](SECURITY.md).

## Licence

[Apache-2.0](LICENSE). Built by [Zealynx Security](https://zealynx.io). Written with Claude Code as an assistant.
