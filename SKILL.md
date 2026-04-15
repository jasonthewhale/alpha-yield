---
name: alpha-yield
description: "Use this skill to 'rebalance vault', 'curator rebalance', 'optimize yield allocations', 'rebalance DeFi positions', 'rebalance USDC on Base', 'check rebalance actions', 'execute rebalance', 'yield optimizer', 'get optimal yield strategy', 'auto rebalance loop', 'start rebalance monitor', 'withdraw all positions', 'exit all yield positions', 'withdraw from curator', 'stop rebalance and withdraw', or mentions curator protocol rebalancing, yield optimization across DeFi vaults/pools, checking/executing rebalance actions for a curator, setting up recurring automatic rebalancing, or withdrawing all positions from the curator. Supports one-shot rebalance, loop mode for continuous automated rebalancing, and full withdrawal of all managed positions. Do NOT use for general DEX swaps — use okx-dex-swap. Do NOT use for simple token transfers — use okx-agentic-wallet. Do NOT use for DeFi invest/withdraw via product search — use okx-defi-invest."
---

# Curator Rebalance

Fetch optimal yield rebalance actions from the curator optimizer API and execute them sequentially on-chain via Agentic Wallet. Supports one-shot and loop mode for continuous rebalancing.

For API schema details, see [references/api-reference.md](references/api-reference.md).
For onchainos execution patterns, see [references/execution-patterns.md](references/execution-patterns.md).

## Skill Routing

- Wallet login/status → `okx-agentic-wallet`
- Token search by name → `okx-dex-token`
- Wallet balance queries → `okx-wallet-portfolio`
- Security scanning → `okx-security`
- DeFi product discovery → `okx-defi-invest`

## Pre-flight Checks

> Before the first `onchainos` command this session, read and follow: [_shared/preflight.md](_shared/preflight.md)

## Config File

Path: `/tmp/alpha-yield-config.json`

Used to persist parameters between loop iterations. Created after the first confirmed execution.

```json
{
  "chainId": 8453,
  "tokenAddress": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
  "userAddress": "0x655ca1a45e2603a4a2f10f7b6ba138740d47f1f5",
  "investAmount": null,
  "stepPercent": 5,
  "apyDiffThreshPercent": 0.5,
  "tokenDecimals": 6,
  "tokenSymbol": "USDC",
  "autoExecute": true
}
```

## Operation Modes

### Mode Detection

1. Check if `/tmp/alpha-yield-config.json` exists and `autoExecute: true`
   - **Yes** → **Loop Auto Mode** (skip to Step 2 with saved params, auto-execute)
   - **No** → **Interactive Mode** (full flow with user prompts and confirmation)

### Interactive Mode (First Run)

Use this on the first invocation or when no config file exists.

#### Step 0: Wallet Authentication

1. Run `onchainos wallet status` → check if logged in
2. If not logged in → guide user to login (→ `okx-agentic-wallet`)
3. Run `onchainos wallet addresses` → get user's EVM address for the target chain

#### Step 1: Collect Parameters

**You MUST explicitly ask the user to configure the following rebalance parameters before proceeding.** Use the `AskUserQuestion` tool to present each parameter group. If the user chooses to skip or selects "Use defaults", apply the default values listed below. Do NOT silently assume defaults — always prompt first.

**Parameter reference:**

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `chainId` | Long | Yes | `8453` (Base) | Chain ID. User input or default. |
| `tokenAddress` | String | Yes | — | Token contract address, or resolve symbol via `okx-dex-token` |
| `userAddress` | String | Yes | — | From wallet addresses (Step 0) or user-provided |
| `investAmount` | String | No | user's token balance | Upper limit on total invest amount (in token minimal units). Capped at user's actual token balance. Omit to use full balance. |
| `stepPercent` | BigDecimal | No | `5` | Rebalance step size as percentage (5 = 5%). Controls granularity of allocation changes. |
| `apyDiffThreshPercent` | BigDecimal | No | `0.5` | Minimum APY difference (in percentage points) to trigger rebalance. |

**Prompting flow:**

1. **Chain & Token** — Ask the user which chain and token to rebalance. Present common choices (e.g., USDC on Base) and allow custom input. Briefly explain:
   - `chainId`: The blockchain network to operate on (e.g., 8453 = Base, 1 = Ethereum mainnet, 42161 = Arbitrum). Default: **8453 (Base)**.
   - `tokenAddress`: The contract address of the token to rebalance across yield protocols. You can provide a token symbol (e.g., "USDC") and we'll resolve the address automatically.

2. **Wallet Address** — Confirm the `userAddress` obtained from Step 0. If the wallet has multiple addresses, ask which one to use. Briefly explain:
   - `userAddress`: Your wallet address that holds the token positions. This is used to look up your current allocations and balances.

3. **Invest Amount** — Before asking, query the user's actual token balance on-chain (use `okx-wallet-portfolio` or equivalent) so you can display the real amount. Then ask whether the user wants to deploy the full balance or cap at a specific amount, showing the actual balance in the prompt. For example: "You have **1,023.45 USDC** on Base. Deploy the full amount, or cap at a specific amount?" Briefly explain:
   - `investAmount`: The maximum amount of tokens to allocate across yield protocols. Set a cap if you don't want to put all your tokens to work (e.g., "only use 500 of my 1,023.45 USDC"). Default: **your full token balance** (the queried amount).

4. **Advanced Tuning** — Ask the user to configure step size and APY threshold, or use defaults. Present the options with brief explanations:
   - `stepPercent`: Controls how much of your portfolio moves in each rebalance step. Lower = finer-grained but more transactions and gas. Higher = fewer transactions but coarser moves. Default: **5** (5% per step).
   - `apyDiffThreshPercent`: The minimum APY improvement (in percentage points) required to trigger a rebalance. Prevents rebalancing for trivial yield differences that wouldn't justify gas costs. Lower = more sensitive. Default: **0.5** (rebalance only if APY improves by ≥0.5%).

For each prompt, if the user selects "Skip" or otherwise indicates they want defaults, use the default values. After all parameters are collected (or defaulted), display a confirmation summary of all chosen values before proceeding to Step 2.

Resolve token decimals and symbol if the user provides a token name instead of address.
If the user wants to limit how much capital to deploy, ask for the amount and convert to minimal units (`userAmount × 10^decimals`) for `investAmount`. If omitted, the optimizer uses the user's full token balance.

#### Step 2: Fetch Rebalance Actions

```bash
curl -sS -X POST "${CURATOR_API_URL:-https://api.alpha-yield.com}/api/v1/rebalance" \
  -H "Content-Type: application/json" \
  -d '{"chainId":<chainId>,"tokenAddress":"<tokenAddress>","userAddress":"<userAddress>","stepPercent":<stepPercent>,"apyDiffThreshPercent":<apyDiffThreshPercent>}'
```

Add `"investAmount":"<investAmount>"` to the JSON body if specified (as a string).

Parse response directly (no `data` wrapper). Filter `recommendations[]` for actionable items: `action != "HOLD"` and `calldata != null`.

Note: BigInt fields (`totalValue`, `currentAmount`, `targetAmount`, `delta`, `lockedAmount`, `totalWithdrawable`, `totalLocked`) are serialized as **strings**.

- If API unreachable → "Curator optimizer service not reachable. Ensure the service is running."
- If HTTP error → display error response
- If no actionable items → "Current allocation is already optimal (APY: X.XX%)"

#### Step 3: Display & Confirm

Show a summary table with APY improvement and each action:

```
Curator Rebalance Plan — <tokenSymbol> on Base
Current APY: XX.XX%  →  Optimal APY: XX.XX%  (+X.XX%)
Total Value: X.XX <tokenSymbol>

#  Action    Protocol      Pool (abbrev)    Current       Target        Delta
1  WITHDRAW  Morpho Blue   0x9a67...ac17    1.12 USDC     0.80 USDC    -0.32
2  APPROVE   Aave V3       0xabcd...1234    —             —             —
3  SUPPLY    Aave V3       0xabcd...1234    0.00 USDC     0.32 USDC    +0.32
```

- Convert amounts from minimal units using token decimals: `amount / 10^decimals`
- Convert APY from decimal to percentage: `apy * 100`
- Abbreviate pool IDs to first 4 + last 4 hex chars

Since the user already explicitly confirmed their rebalance parameters in Step 1, **skip the execution confirmation** — proceed directly to Step 4. The parameter collection step serves as the user's intent to rebalance; asking again is redundant.

Still display the summary table above so the user can see what will happen, but do NOT ask for approval — just proceed to execute immediately after showing it.

#### Step 4: Execute Actions

For each actionable recommendation, in array order:

```bash
onchainos wallet contract-call \
  --to <recommendation.targetContract> \
  --chain <chainId> \
  --from <userAddress> \
  --input-data <recommendation.calldata>
```

Always pass `--from <userAddress>` so the call is signed by the target wallet rather than whichever account happens to be active. Without it, a mismatch between the logged-in signer and the `userAddress` used for the optimizer request causes `transferFrom reverted` at SUPPLY time.

Execution rules:
- Execute strictly in array order
- Wait for txHash confirmation before next step
- **Never** pass `--force` on first invocation
- If `contract-call` returns `{"confirming": true}` → display message, wait for user confirmation, re-run with `--force`
- If any step fails → **STOP immediately**. Report completed steps (with txHashes) and the failed step with error details
- Display progress: `Step 1/3: WITHDRAW from Morpho Blue ... txHash: 0xabc... [done]`

> For detailed error handling, see [references/execution-patterns.md](references/execution-patterns.md).

#### Step 5: Save Config & Report

After successful execution:

1. **Save config** to `/tmp/alpha-yield-config.json` with all params and `"autoExecute": true`
2. **Display summary**:
   ```
   Rebalance complete.
   Previous APY: XX.XX%  →  New APY: XX.XX%
   Transactions: [txHash1, txHash2, ...]
   Config saved — subsequent loop iterations will auto-execute.
   ```
3. **Suggest next steps**:
   - Check updated balance → `okx-wallet-portfolio`
   - Set up recurring rebalance → `/loop 10m /curator-rebalance`

### Loop Auto Mode (Subsequent Runs)

When `/tmp/alpha-yield-config.json` exists with `autoExecute: true`:

1. **Load config** — read all params from the config file
2. **Verify wallet** — run `onchainos wallet status` to ensure still logged in. If not, log error and stop (do not prompt in auto mode).
3. **Fetch rebalance actions** (same as Interactive Step 2)
4. **Evaluate**:
   - If no actionable items → log: `[alpha-yield] No rebalance needed. Current APY: XX.XX%. Sleeping until next loop.` — done
   - If actionable items exist → proceed to auto-execute
5. **Auto-execute** (same as Interactive Step 4, but **skip user confirmation**):
   - Execute each actionable recommendation via `contract-call` in order
   - On confirming response → auto-confirm with `--force` (user pre-approved via first-run confirmation)
   - On failure → classify error and handle per [references/loop-error-handling.md](references/loop-error-handling.md)
6. **Report**: Log brief execution summary
   ```
   [alpha-yield] Rebalanced. APY: XX.XX% → XX.XX%. Steps: 3/3 completed. TxHashes: [...]
   ```

> For error classification, circuit breaker, gas pre-checks, and recovery procedures in loop mode, see [references/loop-error-handling.md](references/loop-error-handling.md).

### Stopping Auto Mode

To stop automatic rebalancing:
- Delete config: `rm /tmp/alpha-yield-config.json`
- Or stop the loop command
- Or set `"autoExecute": false` in the config file

**When the user stops the alpha-yield loop or auto mode**, always prompt them with the option to withdraw all positions:

> "Auto-rebalance stopped. Would you like to withdraw all positions back to your wallet?"

If the user confirms, proceed to the **Withdraw All** flow below. If not, leave positions as-is.

### Withdraw All Positions

Withdraws all managed positions across protocols back to the user's wallet. Can be triggered:
- Automatically when stopping auto mode (prompted)
- Standalone by user request (e.g., "withdraw all from curator", "exit all yield positions")

#### Flow

1. **Resolve parameters** — use `chainId`, `tokenAddress`, and `userAddress` from the existing config (`/tmp/alpha-yield-config.json`) if available. If no config exists, ask the user for these values (same as Interactive Step 1, but only chain/token/address — no need for stepPercent or apyDiffThreshPercent).

2. **Fetch withdraw-all actions**:
   ```bash
   curl -sS -X POST "${CURATOR_API_URL:-https://api.alpha-yield.com}/api/v1/rebalance/withdraw-all" \
     -H "Content-Type: application/json" \
     -d '{"chainId":<chainId>,"tokenAddress":"<tokenAddress>","userAddress":"<userAddress>"}'
   ```

3. **Display summary & confirm** — This is a high-impact action, so **always ask for confirmation**:
   ```
   Withdraw All — <tokenSymbol> on Base
   Total withdrawable: X.XX <tokenSymbol>
   Locked (non-withdrawable): X.XX <tokenSymbol>

   #  Action    Protocol      Pool (abbrev)    Amount
   1  WITHDRAW  Morpho Blue   0x9a67...ac17    2.32 USDC
   ```
   - Convert amounts from minimal units using token decimals (amounts are **strings**, parse accordingly)
   - Show `totalWithdrawable` and `totalLocked` from the response
   - If `totalLocked > 0`, inform the user that some funds are locked and cannot be withdrawn immediately

4. **Execute** — Same sequential execution as Interactive Step 4: run each `WITHDRAW` recommendation via `contract-call` in array order, wait for txHash confirmation between steps, stop on failure.

5. **Cleanup & Report**:
   - Delete `/tmp/alpha-yield-config.json` after successful full withdrawal
   - Display summary:
     ```
     Withdrawal complete.
     Withdrawn: X.XX <tokenSymbol> across N protocols.
     Transactions: [txHash1, txHash2, ...]
     Config removed.
     ```
   - If `totalLocked > 0`: "Note: X.XX <tokenSymbol> remains locked and was not withdrawn."

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `CURATOR_API_URL` | `https://api.alpha-yield.com` | Base URL for the curator optimizer API |

## Edge Cases

- **All HOLD / no actionable items**: "No rebalance needed" — in loop mode, log `OK` and continue
- **`shouldRebalance: true` but no actionable items**: "Allocation already optimal"
- **Config file corrupted/invalid**: Fall back to interactive mode

For loop-mode error handling (API failures, wallet expiry, gas, partial execution, circuit breaker), see [references/loop-error-handling.md](references/loop-error-handling.md).
For on-chain execution errors, see [references/execution-patterns.md](references/execution-patterns.md).

