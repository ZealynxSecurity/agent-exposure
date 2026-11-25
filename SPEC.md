# agent-exposure: specification (M0)

**Status:** draft for M0, 25 Nov 2026. Nothing is implemented yet. M1 (library and CLI) starts 10 Dec 2026.

**One sentence:** agent-exposure tells you **the most an AI agent, or anyone holding its key, can take from your wallet**, per token, across every permission you've given it, and when that stops.

It answers two questions:

1. **Before you grant:** "If I sign this permission, what's the worst that can happen?" You paste the permission request, the signed delegation or the spend-permission struct.
2. **Now:** "What can leave this wallet right now, and through whom?" You give it an address; it reads what's live on chain.

---

## 1. Principles

These are requirements, not aims. Each one has a test in M1.

**P1. Read-only.** agent-exposure holds no private keys, signs nothing, builds no transactions and moves no money. It uses only `eth_call`, `eth_getCode`, `eth_getLogs`, `eth_getBalance` and `eth_blockNumber` against an RPC the user chooses. A bug can give a wrong figure; it can't lose anyone's funds.

**P2. Err on the high side.** Anything it can't fully read is **unbounded**, never zero and never skipped. That covers an unknown caveat, an unknown 7702 delegate, a contract that reverts on a view call, a log range the RPC refused, and a token whose `decimals()` fails. A "bounded" answer must have every input behind it.

**P3. Say what it can't see.** Every report lists its blind spots: off-chain policy engines (Privy, Turnkey and similar), off-chain signatures not yet submitted (Permit2 permits and `permitTransferFrom` signatures, Coinbase `approveWithSignature` permissions, MetaMask delegations the user didn't paste in), chains not scanned, and log ranges not read.

**P4. No backend, no data collection.** No server of ours, no analytics, no accounts. Addresses never leave the user's machine except to the RPC they picked.

**P5. Numbers are exact.** All amounts are integers in the token's base units (`uint256` as decimal strings in JSON). Display formatting is separate and never feeds back into a computation.

**P6. Pinned to a block.** Every report names the chain id, block number and block timestamp it was computed at. "Now" means that block's timestamp, not the local clock.

---

## 2. Model

### 2.1 Grant

A **grant** is one permission that lets one **spender** move one **asset** out of one **owner** account.

| Field | Meaning |
| --- | --- |
| `source` | `erc20-approve`, `permit2`, `coinbase-spend-permission`, `safe-allowance`, `eip7702`, `metamask-delegation` |
| `chainId`, `block` | Where and when it was read |
| `owner` | The account the funds leave |
| `spender` | Who can pull: the agent's key, a contract, or a delegate |
| `asset` | ERC-20 address, or `native` |
| `params` | The source's own fields, as read (amount, period, start, end, spent, nonce, caveat terms) |
| `payee` | `any`, or the list of allowed recipients if the grant restricts them |
| `revoke` | Where the user revokes it: a link or the call to make in their own wallet |

### 2.2 Exposure of one grant

For each grant, agent-exposure computes:

| Figure | Meaning |
| --- | --- |
| `availableNow` | The most the spender can move in one transaction at `block` |
| `window(W)` | The most that can leave within **any** interval of length `W` seconds that starts at or after `block` (default `W` = 86,400) |
| `totalUntilEnd` | The most that can leave from `block` until the grant ends |
| `endsAt` | When the grant stops by itself: a timestamp, or `never` |
| `bounded` | `true` only if all four figures above are finite and fully derived from read data |
| `reasons` | Why a figure is unbounded or flagged (machine-readable codes, section 5) |

Each figure is either an integer or `unbounded`. `window` and `totalUntilEnd` are **upper bounds on what the permission allows**. They ignore the owner's balance (see 2.4).

### 2.3 Aggregation

Grants are grouped by `(chainId, owner, asset, spender)`, then by `(chainId, owner, asset)` across a set of spenders the user names as "the agent's".

- `availableNow`, `window(W)` and `totalUntilEnd` **add up** across grants in a group. Two grants to the same agent are two ways to pull, and both can be used.
- `unbounded` plus anything is `unbounded`.
- `endsAt` of a group is the latest `endsAt` in it (`never` wins).
- **One exception, nested allowances:** a Permit2 allowance only works up to the owner's ERC-20 allowance to the Permit2 contract. Section 3.2 says how that is capped, so the two aren't double-counted.

### 2.4 Balance

The permission figures are the headline. Below them, the report shows the same figures **capped at the owner's current balance** of each asset, labelled "at today's balance". The cap is not a bound: the owner may top the wallet up, and a periodic allowance keeps working on new funds.

---

## 3. Worst-case rules, per source

Notation: `A` is a per-period amount, `P` a period length in seconds, `t` the block timestamp, `W` the window length. A window is the half-open interval `[x, x + W)` for any `x ≥ t`.

**The boundary rule, used by every periodic source.** A periodic allowance refills to `A` at fixed boundaries. A window of length `W` can touch at most `⌈W / P⌉ + 1` periods, so the most that can leave in it is `A × (⌈W / P⌉ + 1)`, less what is already spent if the window starts in the current period, and counting only periods that start before the grant's end. **For `W = P` that is `2A`:** "1 USDC a day" allows 1 USDC one second before a boundary and 1 more at the boundary. This was checked in the source of all four periodic designs below (Coinbase, Safe, MetaMask's period enforcers); streaming caveats don't have it.

All contract details below were read on 25 Nov 2026 at the commits listed in section 11. M1 confirms each against deployed bytecode on a fork before a rule can return `bounded: true`.

### 3.1 ERC-20 `approve`

- Read: `Approval(owner, spender, value)` logs to find candidate spenders, then `allowance(owner, spender)` at `block` for the live value. Logs only point to candidates; the live value always comes from `allowance`.
- `availableNow = window(W) = totalUntilEnd = allowance`. `endsAt = never`.
- Flag `UNLIMITED` when `allowance ≥ 2^255` (many tokens never decrement `type(uint256).max`). The figure stays the exact integer; the text says "unlimited".
- A token whose `allowance` reverts or returns malformed data gives `UNREADABLE` and `unbounded`.
- An approval **to the Permit2 contract** is shown once, as its own line, and caps the Permit2 grants under it (3.2).

### 3.2 Uniswap Permit2 (AllowanceTransfer)

- Read: `allowance(owner, token, spender)` on Permit2 returns `(uint160 amount, uint48 expiration, uint48 nonce)`.
- Permit2 checks `block.timestamp > expiration`, so a grant is live **through** `expiration`. If `t > expiration` it is expired: figures 0, listed as `EXPIRED`. A stored `expiration` of 0 means never set (Permit2 stores `block.timestamp` when 0 is passed), so it also reads as expired.
- Otherwise `availableNow = window(W) = totalUntilEnd = amount`, `endsAt = expiration`.
- `amount = type(uint160).max` is never decremented: flag `UNLIMITED`.
- **Nesting cap:** for each `(owner, token)`, the sum of all Permit2 grants is capped at the owner's ERC-20 `allowance(owner, Permit2)`. The capped figure is what aggregates (2.3), so the ERC-20 line to Permit2 and the Permit2 lines aren't counted twice.
- **Blind spot:** `SignatureTransfer` signatures and unsubmitted `permit` signatures are off chain until used. The report says so whenever the ERC-20 allowance to Permit2 is non-zero.
- Permit2's address (`0x000000000022D473030F116dDEE9F6B43aC78BA3`, the same on every chain we target) is confirmed on chain in M1.

### 3.3 Coinbase `SpendPermissionManager`

A spend permission is the struct `(account, spender, token, uint160 allowance, uint48 period, uint48 start, uint48 end, uint256 salt, bytes extraData)`. `token` is `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for the native token. `start` is inclusive and `end` exclusive.

- Address: `0xf85210B21cC50302F477BA56686d2019dC9b67Ad`, the same on Base, Ethereum, Optimism, Arbitrum and the other chains in its README.
- Read: `SpendPermissionApproved(hash, permission)` and `SpendPermissionRevoked(hash, permission)` logs filtered by `account`, then at `block`: `isApproved(p)`, `isRevoked(p)` and `getCurrentPeriod(p)`, which returns `PeriodSpend(start, end, spend)`.
- `isValid` checks approval and revocation only, **not** the time window. `getCurrentPeriod` reverts when `t < start` or `t ≥ end`. agent-exposure checks the time window itself and treats that revert as "outside the window", never as `UNREADABLE`.
- Periods are fixed, counted from `start`: the current one begins at `t − (t − start) mod P` and ends at the earlier of that plus `P`, and `end`. **The last period is cut short at `end` but still allows the full `A`.** A spend is allowed while `spend ≤ A`.
- Revoked (`isRevoked`), not approved, or `t ≥ end`: figures 0, listed as `REVOKED` or `EXPIRED`. `t < start`: listed as `NOT_YET_ACTIVE`, with figures from `start`.
- `availableNow = A − spend` of the current period.
- `window(W)`: the boundary rule above.
- `totalUntilEnd = (A − spend) + A × (number of later periods whose start is before end)`.
- `endsAt = end − 1`, the last second a spend can happen.
- `payee = any`: the manager sends to the spender, who can forward anywhere.
- **Blind spot:** `approveWithSignature` lets anyone holding the user's signature register a permission later. A signed but unsubmitted permission is invisible on chain until then; the report says so.

### 3.4 Safe allowance module

A delegate can transfer up to `amount` of a token, resetting every `resetTimeMin` minutes.

- Addresses: v0.1.1 at `0xAA46724893dedD72658219405185Fb0Fc91e091C` (Base, Optimism, Arbitrum, Polygon and others), v0.1.0 at `0xCFbFaC74C26F8647cBDb8c5caf80BB5b32E43134` (Ethereum and Base). Base has both, so both are read. The module is only counted if the Safe has it enabled (`isModuleEnabled`).
- Read: `getDelegates(safe, start, pageSize)` (paged), `getTokens(safe, delegate)`, then `getTokenAllowance(safe, delegate, token)`, which returns `[amount, spent, resetTimeMin, lastResetMin, nonce]` with any due reset **already applied** by the view.
- `getTokens` still lists tokens whose allowance was removed; those read as `amount = 0` and are left out.
- `resetTimeMin = 0` is a one-off allowance: all three figures are `amount − spent`, `endsAt = never` (it doesn't expire; it runs out).
- `resetTimeMin > 0`: `P = resetTimeMin × 60` (at most 65,535 minutes, about 45.5 days). Periods are aligned to `lastResetMin + k × resetTimeMin` in whole minutes, so two full spends can be about a second apart across a boundary. `availableNow = amount − spent`; `window(W)` follows the boundary rule with `A = amount`. There is no end date, so `totalUntilEnd = unbounded`, code `NO_END`, and the report shows the rate ("up to `amount` every `resetTimeMin` minutes, for ever").
- A relayer `payment` in the same token counts against the same allowance, so it doesn't raise the figures.
- A delegate removed without clearing its allowances keeps them stored; re-adding the delegate brings them back. Those are listed as `DORMANT`, with figures 0 but the stored amounts shown.
- `payee = any`: the delegate chooses `to`.

### 3.5 EIP-7702 delegation

- Read: `eth_getCode(owner)`. If it is the 23-byte **delegation indicator** `0xef0100 ‖ delegate`, the account runs `delegate`'s code. Fixture F16 confirms that `eth_getCode` returns the indicator on each chain we support.
- The delegate's code runs with the owner's full authority. Unless `delegate` is on agent-exposure's list of **reviewed delegates** (contracts we've read, each with a rule), the result is `unbounded` for **every asset the account holds**, code `UNKNOWN_7702_DELEGATE`, naming the delegate.
- A reviewed delegate's rule says which keys can call what. If a key the agent holds can execute arbitrary calls, the result is still `unbounded`, code `ARBITRARY_EXECUTION`. The reviewed list starts empty in M1.
- `revoke`: re-delegating to the zero address, from the user's wallet. Not every wallet offers it (Rabby does; Revoke.cash can't); the report says which.

### 3.6 MetaMask delegations (ERC-7710, granted through ERC-7715)

Delegations are signed off chain and never registered on chain until redeemed, so they **can't be listed from an address**. They are checked **before granting**, from what the user pastes.

- Input: a delegation, a chain of them (root first), or an ERC-7715 permission request.
- Read on chain: `disabledDelegations(hash)` on the DelegationManager (`0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3`, v1.3.0). A disabled delegation gives 0, listed as `DISABLED`. The delegator can re-enable it, so the report says so.
- Caveats are AND-ed: each must pass on redemption. The bound for an asset is the **minimum** over the caveats that bound it, and `unbounded` if none does. In a chain, each link's caveats apply, so the chain's bound is the minimum over the links.
- Enforcers M2 reads (`terms` layouts as in the source):

| Enforcer | `terms` | Bound |
| --- | --- | --- |
| `ERC20PeriodTransferEnforcer` | token (20) · periodAmount (32) · periodDuration (32) · startDate (32) | Boundary rule, periods aligned to `startDate + k × periodDuration`; no end date of its own |
| `NativeTokenPeriodTransferEnforcer` | periodAmount · duration · startDate (32 each) | The same, for the call's `value` only |
| `MultiTokenPeriodEnforcer` | N × 116 bytes, `address(0)` = native | The same, per token |
| `ERC20TransferAmountEnforcer` | token (20) · max (32) | Lifetime cap `max` |
| `NativeTokenTransferAmountEnforcer` | `uint256` | Lifetime cap on `value` |
| `ERC20StreamingEnforcer` | token · initial · max · perSecond · startTime | `min(initial + perSecond × elapsed, max) − spent`; no boundary doubling |
| `NativeTokenStreamingEnforcer` | initial · max · perSecond · startTime | The same, for `value` |
| `ValueLteEnforcer` | `uint256` | Per call only (`PER_CALL_ONLY` unless something else bounds the total) |
| `TimestampEnforcer` | after (`uint128`) · before (`uint128`), 0 = no limit | Ends the grant at `before` (strict); starts it after `after` |
| `LimitedCallsEnforcer` | `uint256` | Multiplies a per-call cap into a total |
| `AllowedTargetsEnforcer`, `AllowedMethodsEnforcer` | N × 20 bytes, N × 4 bytes | Restrict what can be called; needed before a native-only caveat can be said to bound ERC-20s |

- **A native-token caveat bounds only `value`.** Without target and method caveats, the delegate can call any token's `transfer`, so every ERC-20 the account holds is `unbounded`, code `ARBITRARY_EXECUTION`.
- Enforcer state is keyed by delegation hash, so a delegation never used has spent nothing; figures are computed from `terms` (the streaming enforcer's own `getAvailableAmount` returns 0 before first use, so it isn't relied on).
- **Any enforcer agent-exposure doesn't recognise gives `unbounded`**, code `UNKNOWN_CAVEAT`, with its address. That includes `LogicalOrWrapperEnforcer` until M2 reads it: an OR can loosen a bound.
- Enforcer addresses are matched against MetaMask's published deployments list (section 11), per chain.

---

## 4. Output

### 4.1 Plain text (CLI default)

A daily 1 USDC permission granted from 20 Nov to the end of 2026 (42 periods, so 42 USDC in total at the time of granting), scanned on 10 Dec with 0.60 spent that day:

```
agent-exposure · Base (8453) · block 23,456,789 · 2026-12-10 09:14:03 UTC
Wallet 0xUser…  ·  agent keys: 0xAgent…

USDC
  Coinbase spend permission → 0xAgent…
    now          0.40 USDC  (0.60 of 1.00 spent this period)
    any 24 h     2.00 USDC  ⚠ period boundary: twice the daily figure
    in total    21.40 USDC  until 2026-12-31 23:59:59 UTC (21 more days)
    revoke       Coinbase Wallet → Settings → Spend permissions
  ERC-20 approve → 0xRouter…
    now / any 24 h / in total   unlimited  ⚠ UNLIMITED, no end date
    revoke       revoke.cash, or approve(0xRouter…, 0) from your wallet

  All grants for USDC
    any 24 h     unlimited
    at today's balance (125.00 USDC): 125.00 USDC can leave in 24 h

Not checked: off-chain policy engines; Permit2 and Coinbase signatures not yet submitted;
MetaMask delegations you didn't paste in; chains other than Base.
```

### 4.2 JSON (`--json`, the library's return value, the MCP tool's result)

```json
{
  "schema": "agent-exposure/1",
  "chainId": 8453,
  "block": { "number": 23456789, "timestamp": 1796894043 },
  "owner": "0x…",
  "windowSeconds": 86400,
  "grants": [
    {
      "source": "coinbase-spend-permission",
      "spender": "0x…",
      "asset": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
      "params": { "allowance": "1000000", "period": 86400, "start": 1795132800, "end": 1798761600, "spend": "600000" },
      "availableNow": "400000",
      "window": "2000000",
      "totalUntilEnd": "21400000",
      "endsAt": 1798761599,
      "payee": "any",
      "bounded": true,
      "reasons": ["PERIOD_BOUNDARY_2X", "ANY_PAYEE"],
      "revoke": { "where": "Coinbase Wallet", "how": "Settings → Spend permissions" }
    }
  ],
  "totals": [
    { "asset": "0x8335…2913", "spenders": ["0x…"], "availableNow": "unbounded", "window": "unbounded", "totalUntilEnd": "unbounded", "endsAt": "never", "atBalance": { "balance": "125000000", "window": "125000000" } }
  ],
  "notChecked": ["offchain-policy-engines", "unsubmitted-signatures", "metamask-delegations-not-provided"]
}
```

Amounts are decimal strings of base units; `"unbounded"` is the only non-numeric value. The schema is versioned; a field is never repurposed.

### 4.3 MCP server (M3)

Two read-only tools: `check_permission` (paste a grant, get its exposure) and `scan_wallet` (address, chains, optional agent addresses). No tool signs, sends or revokes.

---

## 5. Reason codes

| Code | Meaning |
| --- | --- |
| `UNLIMITED` | The allowance is at or near the type's maximum |
| `NO_END` | The grant never expires, so its total over time is unbounded |
| `PERIOD_BOUNDARY_2X` | A periodic allowance: `window(P)` is twice the per-period figure |
| `ANY_PAYEE` | Nothing limits where the funds go |
| `UNKNOWN_7702_DELEGATE` | The account is delegated to code we haven't reviewed |
| `ARBITRARY_EXECUTION` | A reviewed delegate lets the key run arbitrary calls |
| `UNKNOWN_CAVEAT` | A delegation caveat whose enforcer we don't recognise |
| `PER_CALL_ONLY` | Only a per-call cap, with no total |
| `UNREADABLE` | A view call reverted, returned malformed data, or the token is non-standard |
| `LOGS_INCOMPLETE` | The RPC refused part of the log range; earlier grants may be missing |
| `NOT_YET_ACTIVE` | The grant starts in the future (figures shown from its start) |
| `EXPIRED`, `REVOKED`, `DISABLED` | Inactive; listed, figures 0 |
| `DORMANT` | A Safe allowance stored for a removed delegate; figures 0, but re-adding the delegate revives it |

---

## 6. Fixtures and tests (M1 and M2)

Every rule in section 3 gets three kinds of test before it can return `bounded: true`:

1. **Recorded on-chain fixtures.** Real grants read at a pinned block on a fork (Anvil), with the expected figures written by hand from the contract source. The list below is filled with block numbers and transaction hashes in M1.
2. **Property tests** of the window maths: for random `(A, P, start, end, spent, t, W)`, a brute-force simulation of an adversarial spender (spend the maximum at every instant allowed) never exceeds the formula, and reaches it.
3. **Mutation tests.** Weakened copies of each rule (drop the `+1` boundary period, read `spent` from the wrong period, treat an unknown caveat as 0, skip the Permit2 cap) must each fail at least one test.

### Fixture list

| # | Source | Case | Expected |
| --- | --- | --- | --- |
| F1 | ERC-20 | Exact allowance, then partly spent | `allowance` read live, not from the log |
| F2 | ERC-20 | `type(uint256).max` approval | `UNLIMITED` |
| F3 | ERC-20 | Approval then `approve(…, 0)` | Not listed as live |
| F4 | ERC-20 | Token without standard events (USDT-style quirks) | `UNREADABLE` or a correct read, never 0 by default |
| F5 | Permit2 | Live allowance with expiry | `amount` until `expiration` |
| F6 | Permit2 | Expired allowance | 0, `EXPIRED` |
| F7 | Permit2 | Permit2 allowance above the ERC-20 allowance to Permit2 | Capped at the ERC-20 allowance |
| F8 | Permit2 | `uint160` max | `UNLIMITED` |
| F9 | Coinbase | Daily permission mid-period, part spent | `availableNow = A − spend`; `window(1 d) = 2A` from the next boundary, as the boundary rule gives |
| F10 | Coinbase | Permission with `end` 42 periods away | `totalUntilEnd` matches a hand count |
| F11 | Coinbase | Revoked permission | 0, `REVOKED` |
| F12 | Coinbase | `start` in the future | `NOT_YET_ACTIVE`; `getCurrentPeriod`'s revert not read as `UNREADABLE` |
| F12b | Coinbase | Last period cut short at `end` | Full `A` still counted for it |
| F13 | Safe | One-off allowance (`resetTimeMin = 0`) | `amount − spent`, no `NO_END` total |
| F14 | Safe | Daily allowance, reset due | `availableNow = amount`, `NO_END` |
| F15 | Safe | Two delegates on one token | Totals add up |
| F15b | Safe | Delegate removed, allowance left stored | `DORMANT` |
| F15c | Safe | Base Safe using v0.1.0 and another using v0.1.1 | Both module addresses read |
| F16 | 7702 | Account delegated to an unknown contract | `unbounded` for every held asset, `UNKNOWN_7702_DELEGATE` |
| F17 | 7702 | Undelegated EOA | No 7702 line |
| F18 | MetaMask | Delegation with an ERC-20 period caveat | 3.3 window rule |
| F19 | MetaMask | Period caveat plus transfer-amount caveat | Minimum of the two |
| F20 | MetaMask | Unknown enforcer address | `UNKNOWN_CAVEAT`, `unbounded` |
| F21 | MetaMask | Disabled delegation | 0, `DISABLED` |
| F22 | MetaMask | Two-link chain, tighter caveat on the second link | Minimum over links |
| F22b | MetaMask | Native period caveat with no target or method caveats | ERC-20s `unbounded`, `ARBITRARY_EXECUTION` |
| F22c | MetaMask | `LogicalOrWrapperEnforcer` | `UNKNOWN_CAVEAT` until M2 reads it |
| F23 | All | RPC rejects a log range | `LOGS_INCOMPLETE`, never a silent gap |
| F24 | All | View call reverts | `UNREADABLE`, `unbounded` |
| F25 | Aggregate | Coinbase + ERC-20 + Safe on one token, one agent | Sum; `unbounded` if any part is |

Cases from Zealynx's agent-money corpus of publicly reported spending-limit failures (a cap that reads the wrong field, a budget that fails open on a null cost, a key change that resets the budget) become extra fixtures where they apply to these sources.

---

## 7. Scope by milestone

| Milestone | What | When |
| --- | --- | --- |
| **M0** | This spec, the gap re-check (section 8) | by 30 Nov 2026 |
| **M1** | TypeScript library and CLI: ERC-20, Permit2, Coinbase, Safe, 7702 on Ethereum and Base; fixtures F1–F17 and F23–F25; property and mutation tests | 10–20 Dec 2026 |
| **M2** | Check before granting: MetaMask delegations and the Coinbase struct, pasted in; F18–F22 | by 31 Dec 2026 |
| **M3** | MCP server; review of the tool by Zealynx's auditors; v1 release | first half of January 2027 |
| Later | Arbitrum and Optimism; Rhinestone/Biconomy Smart Sessions; ZeroDev policies; NEAR and Concordium, reusing our spend-guard work | 2027 |

**Language:** TypeScript on [viem](https://viem.sh), so the same library runs in the CLI, the MCP server and, later, a static web page with no backend.

## 8. The gap, re-checked 25 Nov 2026

The 20 Nov proposal said nobody adds these permissions up or states the worst case. That holds in a **narrower** form than we first wrote. The closest tools, as read on 25 Nov:

| Tool | What it covers | Worst case it shows |
| --- | --- | --- |
| [Revoke.cash](https://github.com/RevokeCash/revoke.cash) | ERC-20, NFT and Permit2 approvals; a Delegations tab lists EIP-7702, Delegate.xyz and Warm.xyz delegations, but [can't revoke 7702 delegations](https://revoke.cash/learn/wallets/what-is-eip7702) | **Value at risk per approval:** the smaller of balance and allowance, times price; the wallet total keeps the largest per token. Its code knows four approval types (ERC-20, ERC-721 single and all, Permit2), so Coinbase spend permissions, Safe allowances and 7702 delegations aren't counted. No time window, no total to an end date |
| [MetaMask Advanced Permissions](https://support.metamask.com/more-web3/dapps/advanced-permissions/) | Lists and revokes ERC-7715 permissions granted through MetaMask | A "total exposure" per permission (for example 25 USDC over a month). We found no mention of the period-boundary case, and nothing totalled across permissions or wallets |
| [Rabby](https://support.rabby.io/en/articles/14150713-how-to-revoke-approvals) | Approvals; checks and revokes 7702 delegations | "Contract Trust Value": what has been spent through a contract, not what can leave |
| [Safe spending limits](https://help.safe.global/en/articles/40842-set-up-and-use-spending-limits) | Allowance-module limits per beneficiary | Remaining amount and reset time per limit; nothing combined |
| Coinbase / Base "Manage permissions" | Lists and revokes spend permissions per app | No worst-case figure found (read from search results only; the help page refused our fetch) |
| [Token Revoke MCP](https://mcp.so/servers/token-revoke-mcp) | ERC-20 approvals | USD at risk per approval (from its listing); revoking needs a private key |

Hackathon projects in the area (Aegis7702, Preflight, agent-leash and others) grant, enforce or grade limits; none we found adds up exposure. Zerion, De.Fi, Blockaid, Wallet Guard, Etherscan and the Biconomy, Rhinestone, ZeroDev, Privy, Turnkey, Crossmint and AgentKit dashboards showed no such feature in our searches, which is not proof that none exists.

**What we dropped from the claim:** "nobody states a worst case". Revoke.cash already caps each approval at the balance and totals the worst case per token, and MetaMask states one permission's total.

**What remains, and is what agent-exposure is for:**
1. **All the ways an agent can pull, in one figure:** Coinbase spend permissions, Safe allowance delegates and 7702 delegations alongside approvals and Permit2, added up per agent and per token.
2. **Time:** the most that can leave in any 24 hours (twice the per-period figure across a boundary; Coinbase's `getCurrentPeriod` uses fixed periods counted from `start`, read in [source](https://github.com/coinbase/spend-permissions/blob/main/src/SpendPermissionManager.sol)), and the total until the end date.
3. **Before granting:** an ERC-7715 request or an ERC-7710 delegation pasted in and bounded, from any wallet.
4. **Erring high:** an unknown 7702 delegate or caveat is "unbounded", not left out.

The closest tool, Revoke.cash, is a revoking tool with a value-at-risk figure. agent-exposure links to it for revoking approvals rather than competing with it.

## 9. Security

agent-exposure is a security tool, so a wrong "bounded" is the bug that matters most. Report one privately through GitHub's private vulnerability reporting on this repo. If the tool finds a real flaw in a vendor's contracts, the vendor hears privately first; the case enters the fixtures only after the fix is public.

## 10. AI use

This spec was written with Claude Code as an assistant, from Zealynx's 20 Nov proposal. The contract details in section 3 were read from the sources in section 11; M1 confirms each against deployed bytecode before it is relied on, as section 6 requires.

## 11. Sources, pinned

| Source | Read at |
| --- | --- |
| Coinbase spend permissions | [`SpendPermissionManager.sol`](https://github.com/coinbase/spend-permissions/blob/e0004e63edc4e17de7aa978293800ac7a16892e5/src/SpendPermissionManager.sol) (struct L29–48, period maths L516–553), [README addresses](https://github.com/coinbase/spend-permissions/blob/e0004e63edc4e17de7aa978293800ac7a16892e5/README.md#L5-L29) |
| Safe allowance module | [`AllowanceModule.sol`](https://github.com/safe-global/safe-modules/blob/9e5f3d0701c4bd5d5fa4ce8ff7d5206f2ccc4acc/modules/allowances/contracts/AllowanceModule.sol) (reset L165–176, views L403–421 and L486–502); deployed v0.1.1 [same reset logic](https://github.com/safe-global/safe-modules/blob/edf7af00279b3af223b012d24064da62247ee843/modules/allowances/contracts/AllowanceModule.sol#L108-L111); [deployments](https://github.com/safe-global/safe-modules-deployments/blob/15d373ba5668f795dbd37181aa0df878f6b85c29/src/assets/allowance-module/v0.1.1/allowance-module.json) |
| Uniswap Permit2 | [`IAllowanceTransfer.sol`](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/interfaces/IAllowanceTransfer.sol#L77-L86), [`AllowanceTransfer.sol`](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L76-L95), [`Allowance.sol`](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/libraries/Allowance.sol#L24-L41) |
| EIP-7702 | [EIP text (Final)](https://github.com/ethereum/EIPs/blob/ade9e9cab7e148bba4cd293f916a4a189cd7a8aa/EIPS/eip-7702.md#L105-L172) |
| MetaMask Delegation Framework | [`src/enforcers/`](https://github.com/MetaMask/delegation-framework/tree/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/enforcers), [`DelegationManager.sol`](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/src/DelegationManager.sol), [deployments](https://github.com/MetaMask/delegation-framework/blob/b4b8c41fc912455a1af2300bfcd1276676b92a2e/documents/Deployments.md#L99-L138) |
| Revoke.cash (gap check) | [repo](https://github.com/RevokeCash/revoke.cash), `packages/core/lib/allowances/index.ts`, read 25 Nov 2026 |
