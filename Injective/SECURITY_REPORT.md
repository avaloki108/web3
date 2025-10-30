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

## Reproduction Steps (SendEnabled=false)

The following steps demonstrate the silent fund loss when bank-layer sends are disabled for a denomination. This can be reproduced on a local Injective node or ephemeral test network.

Prerequisites:
- A local `injectived` node with keys `validator` and a user key `trader`
- Test denom: `inj` (or any bank-tracked denom)
- Exchange module enabled with default subaccounts

1) Start a fresh local chain and fund the user
- Initialize and start `injectived` with default genesis
- Fund `trader` with `100000000000000000inj` (1e17 wei units as needed)

2) Deposit funds into the exchange module
- From `trader`, deposit `1000inj` to the default subaccount via `MsgDeposit` (or the app CLI route you use for exchange deposits). Verify the exchange deposit map reflects the balance and the bank balance for `trader` is reduced accordingly.

3) Disable bank sends for the denom via governance parameter change
- Prepare a gov param-change proposal that sets `bank.params.send_enabled = [{denom:"inj", enabled:false}]`.
- Example command (adjust paths/fees/chain-id as per your setup):

```bash
injectived tx gov submit-proposal param-change proposal.json \
  --from validator --deposit 10000000inj --chain-id local-inj --yes

injectived tx gov vote 1 yes --from validator --chain-id local-inj --yes
injectived tx gov vote 1 yes --from trader --chain-id local-inj --yes
```

Where `proposal.json` contains:

```json
{
  "title": "Disable send for inj",
  "description": "Reproduce exchange withdrawal failure when bank send is disabled",
  "changes": [
    {
      "subspace": "bank",
      "key": "SendEnabled",
      "value": [
        { "denom": "inj", "enabled": false }
      ]
    }
  ]
}
```

4) Attempt a withdrawal from the default subaccount
- From `trader`, withdraw `500inj` from the default subaccount to the bank account using `MsgWithdraw` (or equivalent CLI). Expected behavior: bank send should be rejected by bank params.

5) Observe the mismatch and silent loss
- Actual results with the bug:
  - The integer portion `500inj` is subtracted from the user’s exchange deposit record.
  - The underlying `bankKeeper.SendCoinsFromModuleToAccount` fails due to `SendEnabled=false`.
  - The error is ignored in `SetDepositOrSendToBank`, so the state transition commits.
  - The user’s bank account does NOT receive `500inj`.
  - The exchange module account retains the coins; user-visible accounting shows the withdrawn amount deducted.

6) Optional: Re-enable send and verify funds remain trapped
- Re-enable send for `inj` via a second param change. The user balance does not auto-reconcile; the module still holds the coins, and the deposit record remains reduced.

Telemetry/checks to capture during the run:
- Before and after `MsgWithdraw`, record:
  - Exchange deposit for the default subaccount and denom
  - Bank balance of `trader`
  - Module account balance for the exchange module (sum should increase by the withdrawn integer when bug triggers)

Fix validation (post-patch expectations):
- `MsgWithdraw` should fail atomically when bank send fails; exchange deposit must remain unchanged. Alternatively, if a partial path is required, the code must roll back the deposit decrement on send failure.

## Status
No remediation applied yet. Immediate patch is advised before feature releases or parameter changes that could flip `SendEnabled`.
