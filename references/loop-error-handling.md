# Loop Mode Error Handling

When running in loop auto mode, errors must be handled gracefully to avoid infinite failing loops or silent fund mismanagement.

## Error Classification

### Retryable (continue to next iteration)

| Error | Where in flow | Why retryable |
|---|---|---|
| Curator API timeout | Fetch actions | Network blip; next call likely works |
| Curator API HTTP 5xx | Fetch actions | Server-side transient error |
| Curator API connection refused | Fetch actions | Service restarting |
| RPC node error | Execute txs | Node overloaded, onchainos may failover |
| Broadcast network error | Execute txs | Network hiccup during broadcast |
| Rate limit (429) | Any call | Throttled; loop interval provides natural backoff |

Action: log warning, increment `consecutiveFailures`, continue to next iteration.

### Must Stop (pause auto-execution)

| Error | Where in flow | Why must stop |
|---|---|---|
| Wallet session expired | Wallet check | Can't sign txs; user must re-login |
| Insufficient gas (ETH too low) | Pre-execute check | Can't pay for txs; user must top up |
| Tx simulation failure (`executeResult: false`) | Execute txs | Calldata invalid for current state; needs review |
| On-chain revert | Execute txs | Wastes gas on repeated reverts |
| Partial execution failure | Execute txs (mid-sequence) | Funds in transitional state |
| Config file missing / corrupted | Load config | Can't proceed; fall back to interactive mode |
| Curator API HTTP 4xx | Fetch actions | Bad params; won't self-heal |
| Security scan blocks tx (`risk: block`) | Pre-execute scan | Calldata flagged risky |

Action: set `autoExecute: false` in config, log error with reason. User must fix and re-enable.

## Consecutive Failure Circuit Breaker

Track consecutive failures in the config file:

```json
{
  "consecutiveFailures": 0,
  "lastError": null,
  "lastErrorTimestamp": null
}
```

- On **success** or "no rebalance needed" → reset `consecutiveFailures` to `0`
- On **retryable** error → increment by 1
- On **must-stop** error → immediately set `autoExecute: false` (don't wait for threshold)
- If `consecutiveFailures >= 3` → set `autoExecute: false`, log: `[alpha-yield] PAUSED Circuit breaker: 3 consecutive failures. Last error: <lastError>`

## Gas Balance Pre-check

Before executing on-chain actions, check native ETH balance:

```bash
onchainos wallet balance --chain 8453
```

If ETH balance < `0.001 ETH` → skip execution, pause auto-execution.

## API Timeout Handling

Always use timeouts in loop mode:

```bash
curl -sS --connect-timeout 10 --max-time 30 "${CURATOR_API_URL:-https://api.alpha-yield.com}/api/v1/..."
```

- Timeout → retryable
- Connection refused → retryable
- HTTP 4xx (400 validation error) → must stop

## Stale Calldata Protection

Pool state can change between API fetch and execution. If a transaction reverts:

1. Do NOT retry the same calldata
2. Log the revert reason
3. Next iteration will fetch fresh calldata from the API
4. If the same pool/action reverts **2 consecutive iterations** → pause auto-execution for manual review

## Wallet Session Early Exit

Check wallet status **before** calling the curator API (fail fast):

```bash
onchainos wallet status
```

If not logged in → immediately pause auto-execution. Do not call the API or attempt execution.

## Partial Execution Recovery

If multi-step execution fails partway (e.g., WITHDRAW succeeded but SUPPLY reverted):

1. Log full diagnostic summary (completed txHashes + failed step error)
2. Pause auto-execution — funds may be in transitional state
3. On next interactive run, the optimizer API will see updated on-chain state and generate a fresh plan

## Log Format

```
[alpha-yield] <STATUS> <message>
```

| Status | Meaning |
|---|---|
| `OK` | Successful execution or no rebalance needed |
| `WARN` | Retryable error, will retry next iteration |
| `ERROR` | Must-stop error, auto-execution paused |
| `PAUSED` | Circuit breaker triggered |

Examples:
```
[alpha-yield] OK Rebalanced. APY: 13.74% → 14.12%. Steps: 3/3. TxHashes: [0xabc..., 0xdef...]
[alpha-yield] OK No rebalance needed. Current APY: 14.12%.
[alpha-yield] WARN Curator API timeout. Consecutive failures: 1/3.
[alpha-yield] ERROR Wallet session expired. Auto-execution paused.
[alpha-yield] ERROR Insufficient gas. ETH balance: 0.0002. Pausing.
[alpha-yield] ERROR Partial execution: 1/3 steps. SUPPLY reverted. Pausing.
[alpha-yield] PAUSED Circuit breaker: 3 consecutive failures.
```

