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

The scenario reproduces reliably on a localnet or devnet when bank sends are disabled for the affected denom. The essential sequence is: fund user → deposit to exchange (default subaccount) → disable bank send for denom → withdraw → observe debit without receive.

Prerequisites:
- Running `injectived` node with keys set up and a funded proposer account for governance (or local authority to set `SendEnabled=false`).
- Test denom `inj` or a custom `factory/...` denom.

High-level flow:
1. Create two test accounts: `trader` (user) and `recipient` (default withdrawal target).
2. Fund `trader` with denom `DENOM` (e.g., `inj`). Ensure `recipient` starts at 0 for `DENOM`.
3. `trader` deposits to the exchange module (default subaccount). Verify deposit map reflects the integer portion (fractional truncation is stored separately by keeper logic).
4. Disable bank sends for `DENOM`:
   - Via governance: submit a param change to the `bank` module `send_enabled` list setting `DENOM` to `false`.
   - Or via a privileged `MsgSetSendEnabled` in environments that expose it.
5. From `trader`, attempt a withdrawal of some integer amount `X` of `DENOM` from the default subaccount.
6. Observe results:
   - `bank` transfer from module → account fails due to `SendEnabled=false`.
   - Keeper still subtracts `X` from the deposit map; on-chain deposit decreases by `X`.
   - `recipient` account does not receive funds.
   - Module account balance for `DENOM` retains the funds (they are stuck until manual recovery).
7. Optional: Re-enable send and verify no automatic remediation occurs; user remains debited.

Concrete commands (illustrative):
- Deposit (default subaccount):
  ```bash
  injectived tx exchange deposit --from trader --amount 1000000DENOM --yes
  ```

- Governance proposal to disable send for `DENOM` (param-change JSON example):
  ```json
  {
    "title": "Disable send for DENOM",
    "description": "Temporarily disable bank sends for testing.",
    "changes": [
      {
        "subspace": "bank",
        "key": "SendEnabled",
        "value": {
          "send_enabled": [
            { "denom": "DENOM", "enabled": false }
          ]
        }
      }
    ]
  }
  ```

- Submit and pass proposal using your local governance flow, wait for it to apply.

- Withdraw (this will debit deposit even though send fails):
  ```bash
  injectived tx exchange withdraw --from trader --amount 500000DENOM --yes
  ```

Verification checklist:
- `recipient` balance for `DENOM` remains unchanged after withdrawal.
- Module account balance for `DENOM` has not decreased by the withdrawn amount.
- Exchange deposit map for `trader` (default subaccount) decreased by the withdrawn integer amount `X`.

Notes:
- The bug specifically impacts the default subaccount withdrawal path in `SetDepositOrSendToBank` where the integer truncation and unconditional debit occur; non-default subaccounts follow different transfer paths.
- If your environment lacks easy governance tooling, you can reproduce by configuring a bank send failure through parameterization or a localized fork that rejects sends for `DENOM`.

## Status
No remediation applied yet. Immediate patch is advised before feature releases or parameter changes that could flip `SendEnabled`.
