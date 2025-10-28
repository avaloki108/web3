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
