---
title: "Yield.xyz Plugin"
description: "Yield discovery and position management via the Yield.xyz MCP — the MCP builds unsigned transactions, Base MCP signs and broadcasts them via send_calls."
tags: [yield, staking, lending, vaults]
name: yield-xyz
version: 0.1.0
integration: external-mcp
chains: [base, base-sepolia, ethereum, arbitrum, optimism, polygon, avalanche, bsc]
requires:
  shell: none
  allowlist: []
  externalMcp:
    name: yield-xyz
    transport: http
    url: https://mcp.yield.xyz/mcp
  cliPackage: null
auth: none
risk: [irreversible]
---

# Yield.xyz Plugin

> [!IMPORTANT]
> Run Base MCP onboarding first (see SKILL.md). This plugin also requires the **Yield.xyz MCP** to be connected (see [Detection](#detection) / [Installation](#installation)). The Yield.xyz MCP works anonymously out of the box — no API key or sign-in is needed to start.

## Overview

Yield.xyz is yield infrastructure covering staking, lending, vaults, and tokenized real-world-asset (RWA) products across 80+ networks. The hosted Yield.xyz MCP server discovers and filters yield opportunities, reads positions and rewards, and builds **unsigned transactions** for entering, exiting, and managing positions — it never holds keys, signs, or broadcasts. This plugin pairs it with Base MCP: the Base Account wallet address anchors every Yield.xyz call, and each unsigned transaction the Yield.xyz MCP returns is submitted through Base MCP `send_calls` (or `sign` for message payloads). After broadcasting, the transaction hash is reported back to the Yield.xyz MCP so it can track confirmation. Operate only on the intersection of both systems' chains (the `chains` list above).

## Detection

The Yield.xyz MCP advertises tools named `yields_get_all`, `yields_get`, `yields_get_balances`, `actions_enter`, `actions_exit`, `actions_manage`, `submit_hash`, and `get_transaction` (plus supporting discovery tools). If none of these are exposed in the current session, the MCP isn't connected → see [Installation](#installation). Do not fall back to calling Yield.xyz HTTP endpoints directly — the MCP is the supported path, and its tool catalog (read at runtime) is the source of truth for parameters and semantics.

## Installation

The Yield.xyz MCP is a hosted remote server at `https://mcp.yield.xyz/mcp` (streamable HTTP). Register it once in the harness's MCP config:

**Claude Code**

```bash
claude mcp add yield-xyz --transport http https://mcp.yield.xyz/mcp
```

**Cursor / JSON-config harnesses**

```json
{
  "mcpServers": {
    "yield-xyz": {
      "url": "https://mcp.yield.xyz/mcp"
    }
  }
}
```

For clients without native remote-MCP support, bridge over stdio:

```json
{
  "mcpServers": {
    "yield-xyz": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.yield.xyz/mcp"]
    }
  }
}
```

**Claude.ai / ChatGPT** — add a custom connector pointing at `https://mcp.yield.xyz/mcp`. No OAuth or API key is required.

## Surface Routing

All protocol logic lives behind the two MCPs, so routing is uniform wherever both are connected.

| Capability | Surface | Execution Path |
|---|---|---|
| Discover yields, read positions/rewards, network/provider lookups | Any surface with the Yield.xyz MCP connected | Yield.xyz MCP tools directly |
| Build enter/exit/manage transactions | Any surface with the Yield.xyz MCP connected | Yield.xyz MCP `actions_*` → unsigned transactions |
| Sign + broadcast | Any surface with Base MCP | Base MCP `send_calls` (batched calls) or `sign` (message payloads) |
| Record result + confirmation | Any surface with the Yield.xyz MCP connected | Yield.xyz MCP `submit_hash` → `get_transaction` polling |
| Any capability, Yield.xyz MCP not connectable (harness lacks MCP connector support) | Chat-only surface | **Stop.** Tell the user the integration requires connecting the Yield.xyz MCP (see [Installation](#installation)). Do not improvise a `web_request` workaround. |

## Orchestration

The wallet address comes from Base MCP `get_wallets` (`baseAccount.address`; an agent wallet also works when its `inSession` is `true`). Read the Yield.xyz MCP's own tool descriptions at runtime for full parameter semantics — the steps below are the routing skeleton.

### Enter a position

1. `get_wallets` (Base MCP) → wallet address; confirm the target chain is in `supportedChains`.
2. Optionally `get_portfolio` (Base MCP) → feed non-zero holdings into the Yield.xyz discovery filters so only yields the wallet can actually fund are surfaced.
3. `yields_get_all` (Yield.xyz MCP) with the user's token/network filters → present options neutrally; `yields_get` for detail on the chosen one. If the yield requires validator selection, list validators via the MCP and have the user choose.
4. For permissioned RWA yields (`kycRequired: true`), check the wallet's eligibility via the MCP's KYC-status tool first; if not approved, send the user to the returned onboarding URL and stop.
5. Confirm amount and terms (lockup, cooldown, fees) with the user.
6. `actions_enter` (Yield.xyz MCP) → an action containing `transactions[]`, each with an `unsignedTransaction`.
7. Submit via Base MCP — see [Submission](#submission).
8. `submit_hash` (Yield.xyz MCP) for **every** `transactionId` in the action (mandatory), then poll `get_transaction` until a terminal status (`CONFIRMED` / `FAILED` / `SKIPPED`). On `FAILED`, stop and report — never sign later steps.

### Exit a position

1. `yields_get` → confirm exits are open; surface any cooldown/unbonding period to the user before proceeding.
2. `actions_exit` (Yield.xyz MCP) → unsigned transactions.
3. Submit via Base MCP, then `submit_hash` + `get_transaction` polling as above. Some exits are multi-step and async (e.g. unstake now, withdraw after cooldown) — the follow-up step appears later as a pending action on the balance.

### Check balances / claim rewards

1. `yields_get_balances` (Yield.xyz MCP) with the wallet address → positions, rewards, and any `pendingActions` (e.g. `CLAIM_REWARDS`, `WITHDRAW`).
2. To act on one, `actions_manage` with the pending action's `type` and `passthrough` values from the balance response.
3. Submit via Base MCP, then `submit_hash` + `get_transaction` polling as above.

## Submission

The target Base MCP tool is **`send_calls`** (and **`sign`** for message payloads).

Map **every** transaction in the action's `transactions[]` (in `stepIndex` order) into the `calls` array of **one** `send_calls` — a field copy only, never changing any value:

- `chain` — the yield's network mapped to the Base MCP chain string. Slugs differ for two chains: Yield.xyz `avalanche-c` → Base MCP `avalanche`, and Yield.xyz `binance` → Base MCP `bsc`. All other supported slugs (`base`, `base-sepolia`, `ethereum`, `arbitrum`, `optimism`, `polygon`) match as-is.
- `calls[]` — one entry per transaction: `to` → the transaction's `to`, `data` → its calldata hex, `value` → its value as **hex wei** (`0x0` if absent).
- Omit `gas`, `nonce`, and `from` — the Base Account fills those and signs.

The Base Account is a smart wallet, so the batch (e.g. ERC-20 approval + deposit) executes **atomically with a single user approval** and produces **one** on-chain hash — see [batch-calls.md](../references/batch-calls.md). Only batch transactions that exist together: if the action is async and multi-step, execute what's available now, wait for confirmation, then fetch and sign the next step — never batch across that boundary.

If a transaction is flagged as a message (`isMessage: true`, or an EIP-712 typed-data payload), use `sign` instead — `type: personal_sign` with the message, or `type: typed_data` with the payload. Messages can't be batched with calls.

`send_calls` / `sign` return an `approvalUrl` + `requestId`; present the approval link neutrally and poll `get_request_status` until `completed` (capture the hash) or `failed` (do not retry with modified values) — see [approval-mode.md](../references/approval-mode.md). Because the batch yields one hash covering multiple Yield.xyz `transactionId`s, call `submit_hash` once per `transactionId` with that same hash.

## Example Prompts

**"What can I earn on the USDC in my wallet?"**

1. `get_wallets` → address; `get_portfolio` → confirm USDC holdings and chains.
2. `yields_get_all` (Yield.xyz MCP) filtered to USDC on the wallet's chains → present a neutral comparison table (rate, TVL, lockup, provider).
3. Read-only — nothing to submit.

**"Stake 0.5 ETH with a liquid staking provider on Ethereum"**

1. `get_wallets` → address; confirm `ethereum` is supported and the wallet holds ≥ 0.5 ETH (`get_portfolio`).
2. `yields_get_all` → ETH staking options on Ethereum; user picks one; `yields_get` for terms.
3. Confirm amount + terms → `actions_enter` → unsigned transaction(s).
4. `send_calls` (one atomic batch) → user approves → `get_request_status` → hash.
5. `submit_hash` per `transactionId` → poll `get_transaction` to `CONFIRMED`.

**"Claim my staking rewards"**

1. `get_wallets` → address; `yields_get_balances` → positions with `pendingActions`.
2. For the claimable one: `actions_manage` with its `type` + `passthrough` → unsigned transaction.
3. `send_calls` → approve → `submit_hash` → poll to `CONFIRMED`.

**"Withdraw my USDC from that vault"**

1. `yields_get` → confirm exit is open; surface any cooldown before proceeding.
2. `actions_exit` → unsigned transaction(s) → `send_calls` → approve.
3. `submit_hash` + poll. If the exit is two-phase, tell the user a second step (withdraw after cooldown) will appear later under `pendingActions`.

## Risks & Warnings

- **`irreversible`** — Entering a yield can lock funds behind warmup, cooldown, or unbonding periods (days to weeks for some staking products), and on-chain deposits can't be undone once submitted. Before requesting any signature: surface the yield's lockup/cooldown terms and minimum/maximum entry limits, confirm the exact amount with the user, and never proceed silently. If any value in a built transaction looks wrong, do not sign — have the Yield.xyz MCP build a **new** action instead.
- **Never modify an `unsignedTransaction`** returned by the Yield.xyz MCP — not addresses, amounts, fees, or encoding. Copy fields verbatim into `send_calls`. If the amount is wrong, rebuild the action; modifying transaction bytes can permanently lose funds.

## Notes

- **Access modes.** The Yield.xyz MCP works anonymously with a free-tier quota on four metered tools (balances and the three `actions_*` tools; currently 30 calls per wallet per tool per 24h). On a `Free-tier quota exceeded` error, surface the error verbatim (it includes the retry window) and let the user choose: wait, provide a Yield.xyz API key, or pay per call over **x402** — Base MCP settles that via `initiate_x402_request` → user approval → `complete_x402_request` against `https://mcp.yield.xyz/x402/<tool>`. Never silently retry.
- **Chain coverage.** Yield.xyz spans 80+ networks; this plugin uses only those Base MCP can sign on (the `chains` list). For yields on other networks, tell the user the network isn't reachable from Base MCP rather than improvising.
- **Transaction terminal states.** `get_transaction` statuses end at `CONFIRMED`, `FAILED`, or `SKIPPED`. `SKIPPED` means the step needed no on-chain transaction — proceed to the next step.
- **RWA eligibility.** Permissioned RWA yields gate deposits by wallet allowlist/KYC. A deposit from a non-eligible wallet can revert on-chain, so always check eligibility via the MCP before building an enter for a `kycRequired` yield.
- **Neutral presentation.** Present yield options as data (rate, TVL, lockup, provider, risk fields when present) and let the user choose — don't recommend or default to a specific protocol or token.
