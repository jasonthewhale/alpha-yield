# Curator Optimizer API Reference

## Endpoint

```
GET /priapi/v1/invest/activity/test/optimizer/rebalance
```

Base URL: `${CURATOR_API_URL:-https://beta.okex.org:443}`

## Query Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `chainId` | Long | Yes | — | Chain ID (e.g., `8453` for Base) |
| `tokenAddress` | String | Yes | — | Token contract address to rebalance |
| `userAddress` | String | Yes | — | User's wallet address holding the positions |
| `investAmount` | String | No | user's token balance | Upper limit on total invest amount, in token minimal units (e.g., `500000000` = 500 USDC). Capped at user's actual token balance. Omit to use full balance. |
| `stepPercent` | BigDecimal | No | `5` | Rebalance step size as percentage (5 = 5%). Controls granularity of allocation changes. Lower = finer steps, more transactions. |
| `apyDiffThreshPercent` | BigDecimal | No | `0.5` | Minimum APY difference (in percentage points) to trigger a rebalance. Lower = more sensitive. |

## Example Requests

```bash
# Rebalance existing positions only (use defaults)
curl -sS "${CURATOR_API_URL:-https://beta.okex.org:443}/priapi/v1/invest/activity/test/optimizer/rebalance?chainId=8453&tokenAddress=0x833589fcd6edb6e08f4c7c32d4f71b54bda02913&userAddress=0x655ca1a45e2603a4a2f10f7b6ba138740d47f1f5"

# Rebalance with additional 100 USDC capital and custom thresholds
curl -sS "${CURATOR_API_URL:-https://beta.okex.org:443}/priapi/v1/invest/activity/test/optimizer/rebalance?chainId=8453&tokenAddress=0x833589fcd6edb6e08f4c7c32d4f71b54bda02913&userAddress=0x655ca1a45e2603a4a2f10f7b6ba138740d47f1f5&investAmount=100000000&stepPercent=3&apyDiffThreshPercent=0.3"
```

## Response Schema

```json
{
  "code": 0,
  "msg": "",
  "error_code": "0",
  "error_message": "",
  "detailMsg": "",
  "data": {
    "tokenAddress": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "recommendations": [
      {
        "protocol": "Morpho Blue",
        "poolId": "0x9a6703389df0f8e9106c9f9c840bd0812a083bd9689f5e1cfe985780b54fac17",
        "action": "HOLD",
        "currentAmount": 1123585,
        "targetAmount": 1123585,
        "delta": 0,
        "lockedAmount": 0,
        "supplyApy": 0.13736008812735867,
        "targetContract": null,
        "calldata": null
      }
    ],
    "currentGlobalApy": 0.13736008263454647,
    "optimalGlobalApy": 0.13736008263454647,
    "totalValue": 1123585,
    "protocolCount": 1,
    "poolCount": 25,
    "shouldRebalance": true
  }
}
```

## Response Fields

### Top-level

| Field | Type | Description |
|---|---|---|
| `code` | integer | `0` = success, non-zero = error |
| `msg` | string | Error message (empty on success) |
| `error_code` | string | Error code string |
| `error_message` | string | Detailed error message |
| `data` | object | Response payload |

### `data` Object

| Field | Type | Description |
|---|---|---|
| `tokenAddress` | string | The token being rebalanced |
| `recommendations` | array | List of rebalance recommendations (see below) |
| `currentGlobalApy` | float | Current weighted APY across all positions (decimal, e.g., 0.1374 = 13.74%) |
| `optimalGlobalApy` | float | Projected APY after rebalancing (decimal) |
| `totalValue` | integer | Total portfolio value in token minimal units |
| `protocolCount` | integer | Number of protocols analyzed |
| `poolCount` | integer | Number of pools analyzed |
| `shouldRebalance` | boolean | Whether any rebalance is possible |

### `recommendations[]` Object

| Field | Type | Description |
|---|---|---|
| `protocol` | string | Protocol name (e.g., "Morpho Blue", "Aave V3") |
| `poolId` | string | Pool/market identifier |
| `action` | string | Action enum: `HOLD`, `WITHDRAW`, `APPROVE`, `SUPPLY` |
| `currentAmount` | integer | Current allocation in token minimal units |
| `targetAmount` | integer | Target allocation after rebalance |
| `delta` | integer | Change amount (`targetAmount - currentAmount`) |
| `lockedAmount` | integer | Amount locked/non-movable in the protocol |
| `supplyApy` | float | Pool's current supply APY (decimal) |
| `targetContract` | string\|null | Contract address for execution (null for HOLD) |
| `calldata` | string\|null | Hex-encoded calldata for execution (null for HOLD) |

## Action Enums

| Action | Description | Has Calldata | Execution Required |
|---|---|---|---|
| `HOLD` | Current allocation is optimal, no change | No (null) | No |
| `WITHDRAW` | Withdraw tokens from a protocol pool | Yes | Yes |
| `APPROVE` | ERC-20 token approval for the target protocol's contract | Yes | Yes |
| `SUPPLY` | Deposit/supply tokens into a protocol pool | Yes | Yes |

## Amount Units

- `currentAmount`, `targetAmount`, `delta`, `lockedAmount`, `totalValue`: **token minimal units**
  - USDC (6 decimals): 1123585 = 1.123585 USDC
  - WETH (18 decimals): 1000000000000000000 = 1.0 WETH
- `supplyApy`, `currentGlobalApy`, `optimalGlobalApy`: **decimal fraction**
  - 0.1374 = 13.74% APY
  - Multiply by 100 and append `%` for display

## Parameter Tuning

### `stepPercent`
Controls the granularity of rebalance steps as a percentage. Lower values produce finer-grained reallocations but may generate more transactions.
- `5` (default): Move up to 5% of total value per step
- `3`: Finer steps, more transactions
- `10`: Coarser steps, fewer transactions

### `apyDiffThreshPercent`
Minimum APY improvement threshold (percentage points) to trigger rebalancing. Prevents rebalancing for trivial yield differences that wouldn't justify gas costs.
- `0.5` (default): Rebalance only if APY improves by >=0.5 percentage points
- `0.3`: More sensitive, triggers on smaller improvements
- `1`: Less sensitive, only triggers on significant improvements

### `investAmount`
Upper limit on total invest amount in token minimal units. The optimizer will allocate up to this amount across pools. If not provided, defaults to the user's full token balance. If the provided value exceeds the user's balance, it is capped at the actual balance.
- Omitted (default): Use user's full token balance as investable amount
- `500000000`: Cap total investment at 500 USDC (6 decimals), even if balance is higher
- Useful for: partial deployment ("only put 500 of my 1000 USDC to work")

---

## Withdraw All Endpoint

```
GET /priapi/v1/invest/activity/test/optimizer/withdrawAll
```

Generates withdrawal actions for all positions managed by the curator, returning funds to the user's wallet.

### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `chainId` | Long | Yes | Chain ID (e.g., `8453` for Base) |
| `tokenAddress` | String | Yes | Token contract address |
| `userAddress` | String | Yes | User's wallet address holding the positions |

### Example Request

```bash
curl -sS "${CURATOR_API_URL:-https://beta.okex.org:443}/priapi/v1/invest/activity/test/optimizer/withdrawAll?chainId=8453&tokenAddress=0x833589fcd6edb6e08f4c7c32d4f71b54bda02913&userAddress=0x655Ca1A45E2603a4A2F10f7b6Ba138740D47F1F5"
```

### Response Schema

```json
{
  "code": 0,
  "msg": "",
  "error_code": "0",
  "error_message": "",
  "detailMsg": "",
  "data": {
    "tokenAddress": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "recommendations": [
      {
        "protocol": "Morpho Blue",
        "poolId": "0x9a6703389df0f8e9106c9f9c840bd0812a083bd9689f5e1cfe985780b54fac17",
        "action": "WITHDRAW",
        "currentAmount": 2323605,
        "targetAmount": 0,
        "delta": -2323605,
        "lockedAmount": 0,
        "supplyApy": 0,
        "targetContract": "0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb",
        "calldata": "0x5c2bea49..."
      }
    ],
    "totalWithdrawable": 2323605,
    "totalLocked": 0
  }
}
```

### Response Fields (data)

| Field | Type | Description |
|---|---|---|
| `tokenAddress` | string | The token being withdrawn |
| `recommendations` | array | List of WITHDRAW actions (same schema as rebalance recommendations) |
| `totalWithdrawable` | integer | Total amount that can be withdrawn, in token minimal units |
| `totalLocked` | integer | Total amount locked across protocols that cannot be withdrawn immediately |

All `recommendations[]` entries will have `action: "WITHDRAW"` and `targetAmount: 0`. Execute them sequentially via `contract-call` in array order.

---

## Error Handling

| Scenario | Indicator | Action |
|---|---|---|
| API unreachable | Connection refused / timeout | Check if the curator service is running at the configured URL |
| `code` != 0 | Non-zero `code` field | Display `error_message` to user |
| Empty `recommendations` | Empty array | Inform user: "No pools found for this token on this chain" |
| All HOLD | Every recommendation has `action: "HOLD"` | Inform user: "Current allocation is already optimal" |
