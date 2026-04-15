# Execution Patterns — onchainos contract-call on Base

## contract-call Command Reference

```bash
onchainos wallet contract-call \
  --to <contract_address> \
  --chain <chainIndex> \
  --from <user_address> \
  --input-data <hex_calldata>
```

| Parameter | Required | Description |
|---|---|---|
| `--to` | Yes | Target contract address |
| `--chain` | Yes | Numeric chain ID (Base = `8453`) |
| `--from` | Yes | Sender address — must match the `userAddress` passed to the optimizer. Prevents signer/userAddress mismatches when multiple accounts/addresses exist. |
| `--input-data` | Yes | Hex-encoded calldata from the optimizer API |
| `--force` | No | Skip confirmation prompt. **NEVER on first invocation.** |

> **Note**: Do NOT pass `--value`. The CLI does not support it. Curator rebalance actions are ERC-20 token interactions (approve/supply/withdraw), not native token transfers.

## Sequential Execution Protocol

Execute each actionable recommendation strictly in array order:

```
for each recommendation where action ∈ {WITHDRAW, APPROVE, SUPPLY} and calldata != null:
    1. Execute contract-call with recommendation's targetContract and calldata
    2. Wait for txHash response (contract-call handles signing + broadcast internally)
    3. Confirm success before proceeding to next step
    4. If failure → STOP, do not execute remaining steps
```

Rules:
- **Never parallelize** — each step may depend on the previous (e.g., APPROVE before SUPPLY)
- **Wait for confirmation** — `contract-call` returns txHash on success
- **Stop on failure** — report completed steps and the failed step

## Confirming Response Handling

Some commands return a confirming response (exit code 2) when backend requires user confirmation:

```json
{
  "confirming": true,
  "message": "Human-readable prompt to show the user.",
  "next": "Instructions for what to do after user confirms."
}
```

Flow:
1. Display the `message` field to the user
2. If user confirms → re-run the same command with `--force` appended
3. If user declines → cancel and do NOT proceed to remaining steps

**CRITICAL**: Never pass `--force` on the first invocation. Only add `--force` after receiving a confirming response AND getting explicit user confirmation.

## Error Recovery

| Error Type | Indicator | Action |
|---|---|---|
| Simulation failure | `executeResult: false` | Display `executeErrorMsg`, do NOT broadcast. Stop remaining steps. |
| Gas estimation failure | Gas-related error | Suggest checking Base chain ETH balance for gas. |
| Broadcast failure | Network/broadcast error | Report error. Do NOT auto-retry. |
| On-chain revert | Tx reverted | Report revert reason if available. Stop remaining steps. |
| Confirming required | Exit code 2, `confirming: true` | Show message to user, wait for confirmation, re-run with `--force`. |
| Wallet not logged in | Auth error | Direct user to `okx-agentic-wallet` for login. |

## Diagnostic Summary Template

On failure, report the following:

```
Rebalance Execution Summary
============================
Step 1/3: WITHDRAW  from Morpho Blue    txHash: 0xabc123...  [done]
Step 2/3: APPROVE   for Aave V3         txHash: 0xdef456...  [done]
Step 3/3: SUPPLY    into Aave V3        [FAILED]

Error Details:
  step:          3 of 3
  actionType:    SUPPLY
  protocol:      Aave V3
  contract:      0x1234...5678
  chain:         Base (8453)
  errorMessage:  <human-readable error>
  timestamp:     2024-03-15T10:30:00Z

Steps 1-2 completed successfully. Step 3 failed.
Funds from WITHDRAW (step 1) may need manual reallocation.
```

## Security Considerations

- **Transaction simulation**: `contract-call` runs pre-execution simulation automatically. If simulation fails, the tx is NOT broadcast.
- **TEE signing**: Private keys never leave the secure enclave.
- **Optional security scan**: Before executing, consider running `onchainos security tx-scan` on each calldata:
  ```bash
  onchainos security tx-scan --chain 8453 --from <userAddress> --to <targetContract> --data <calldata>
  ```
  If the scan returns a risk action of `block`, do NOT proceed.
