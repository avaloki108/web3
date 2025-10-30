# Injective Core Security Review – Deposit Withdrawal Bug

## Overview
During a focused audit of the exchange module (commit `f1e149e` / `dev` branch), I identified a critical flaw that lets withdrawals from default subaccounts silently lose user funds whenever the bank transfer path is unavailable. The issue lives in `injective-chain/modules/exchange/keeper/deposit.go:222`.

## Vulnerability: Silent Withdrawal Loss on Bank Send Failure
- **Component**: Exchange keeper, `SetDepositOrSendToBank`
- **Description**: When default subaccounts withdraw, the keeper truncates the integer portion of the deposit, attempts `bankKeeper.SendCoinsFromModuleToAccount`, and regardless of the result subtracts that integer from the on-chain deposit map. The returned error is ignored (`_ = ...`).
- **Exploit Path**:
  1. Deposit funds into a non-default subaccount.
  2. Trigger or wait for a temporary `MsgSetSendEnabled` disablement (or any bank-level failure) for the denom.
  3. Initiate `MsgWithdraw`/`MsgExternalTransfer` to the default subaccount.
  4. The send fails, the deposit is debited anyway, and the user never receives their coins.
  5. When the denom is re-enabled the funds remain trapped in the module account; the ledger shows the user has already withdrawn them.
- **Impact**: Direct, unrecoverable user fund loss on affected withdrawals. Severity: Critical.
- **Environment Dependencies**: Requires a bank-layer send failure (which governance or operational errors can trigger). No special permissions beyond standard withdrawal access.

## Recommendations
1. Propagate the send error and abort the state transition, or explicitly roll back the deposit adjustments whenever the send fails.
2. Add integration tests for default-subaccount withdrawals under `SendEnabled=false` conditions.
3. Include an invariant that enforces module balances match the sum of stored deposits to catch similar regressions.

## Status
No remediation applied yet. Immediate patch is advised before feature releases or parameter changes that could flip `SendEnabled`.

## Reproduction steps (SendEnabled=false)

These steps demonstrate the silent loss when bank sends are disabled for a denom via governance. They assume a local Injective node using standard Cosmos SDK governance flows. Adjust chain-id, key names, and fees as needed.

1) Prerequisites
- A running localnet (`injectived start`) with at least one funded validator and one user key: `val` and `user`
- Denom to test: `inj` (or another bank-registered denom in your setup)

2) Disable bank sends via param-change proposal
- Create `proposal.json` to set `send_enabled=false` for the target denom:

```json
{
  "title": "Disable bank sends for INJ",
  "description": "Temporarily disable sends for INJ to reproduce exchange withdrawal bug.",
  "changes": [
    {
      "subspace": "bank",
      "key": "SendEnabled",
      "value": [
        { "denom": "inj", "enabled": false }
      ]
    }
  ],
  "deposit": "10000000inj"
}
```

- Submit and pass the proposal:

```bash
injectived tx gov submit-proposal param-change proposal.json \
  --from val --chain-id local-injective --gas auto --gas-adjustment 1.4 -y

# Vote yes and wait for it to pass (or shorten voting period in genesis for localnet)
injectived tx gov vote 1 yes --from val --chain-id local-injective -y
```

- Verify sends are disabled for `inj`:

```bash
injectived q bank send-enabled inj
```

3) Prepare exchange deposits
- Ensure `user` has `inj` balance:

```bash
injectived q bank balances $(injectived keys show user -a)
```

- Deposit into exchange from a non-default subaccount (or create/choose one). Commands vary by current CLI; the effect needed is: user has a positive integer deposit balance recorded in the keeper for the default subaccount path to withdraw from.

4) Attempt withdrawal to default subaccount while sends are disabled
- Trigger a withdrawal from the exchange for the integer amount portion. Under the hood, the keeper will:
  - Truncate to integer coins
  - Call `bankKeeper.SendCoinsFromModuleToAccount` to the user’s default account
  - Ignore any returned error
  - Decrement the stored deposit regardless

5) Observe the silent loss
- Expected outcomes:
  - The bank send fails due to `send_enabled=false` (observe in node logs)
  - The user’s bank balance does NOT increase
  - The exchange deposit balance decreases by the withdrawn integer amount
  - The module account retains the coins (sum of module balances > sum of user deposits)

Suggested verifications
- Compare before/after:

```bash
# Before and after balances
injectived q bank balances $(injectived keys show user -a)

# Module account balance (replace with actual exchange module address)
injectived q bank balances <exchange_module_account_bech32>

# Exchange deposit state (CLI/GRPC endpoint for subaccount deposits)
# Confirm user deposit decreased despite failed bank send
```

6) Re-enable sends and confirm mismatch persists
- Re-run a param-change to set `send_enabled=true` for `inj` and note that the previously “withdrawn” coins are not automatically credited to the user; they remain in the module account while the ledger shows the user already withdrew them.

Notes
- The precise CLI for deposit/withdraw may differ by release; the bug is in the keeper at `SetDepositOrSendToBank` and triggers whenever the bank send path returns an error.
- A unit/integration test can simulate this by stubbing `bankKeeper.SendCoinsFromModuleToAccount` to return an error and asserting the deposit map is still decremented.
