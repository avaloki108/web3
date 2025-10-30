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

## Reproduction steps (localnet) with SendEnabled=false

Pre-requisites:
- A local Injective node (devnet/localnet) with the exchange module enabled
- Two funded accounts: `trader` (user) and `gov` (governance authority)
- A test denom, for example `inj`, with balances in `trader`

High level flow:
1) Deposit into a non-default subaccount.
2) Disable bank sends for the denom via governance (`MsgSetSendEnabled{denom:"inj", send_enabled:false}`).
3) Withdraw from the non-default subaccount to the default subaccount.
4) Observe: bank send fails silently; deposit mapping decreases; module balance increases; user wallet does not receive funds.

Concrete steps:
1. Create a non-default subaccount for `trader` and deposit funds
   - Derive a subaccount ID (any non-default). Example pseudo:
     - Subaccount: `subacc = SubaccountID(trader, nonce=1)`
   - Deposit funds from `trader` into `subacc` using the exchange deposit message:
     - Tx: `MsgDeposit{ sender: trader, subaccount_id: subacc, amount: 10_000_000inj }`

2. Disable `inj` sends via governance or authority
   - Submit parameter change to bank:
     - Tx: `MsgSetSendEnabled{ authority: gov, entries: [{denom: "inj", send_enabled: false}] }`
   - Wait for the proposal (if governance) to pass or for the tx to be included (if direct auth in localnet).
   - Verify using a dry-run wallet transfer that `bank` rejects sends of `inj`.

3. Attempt a withdrawal from the non-default subaccount
   - Tx: `MsgWithdraw{ sender: trader, subaccount_id: subacc, amount: 9_000_000inj }`
   - Under the hood, the keeper truncates the integer portion and calls:
     - `bankKeeper.SendCoinsFromModuleToAccount(exchange_module, default_wallet_of(trader), amount)`
   - Due to `send_enabled=false`, the bank send returns an error.

4. Observe the faulty state transition
   - Expected (correct) behavior: abort state change and leave deposit untouched.
   - Actual behavior (bug): deposit mapping decreases by the integer amount, but the bank send error is ignored; funds never reach the default wallet.
   - Validate with queries:
     - Query deposits for `subacc`: decreased by ~9_000_000inj
     - Query `trader` default wallet: unchanged balance
     - Query exchange module account balance: increased by ~9_000_000inj

5. Optional: Re-enable sends and confirm funds remain stuck
   - Tx: `MsgSetSendEnabled{ authority: gov, entries: [{denom: "inj", send_enabled: true}] }`
   - Even after re-enable, the exchange module retains the extra balance; the ledger already debited the user deposit. No automatic refund occurs.

Notes:
- The bug is triggered specifically when withdrawing from a non-default subaccount to the default wallet path that uses `SetDepositOrSendToBank`.
- Any bank-layer failure (not only `send_enabled=false`) that makes `SendCoinsFromModuleToAccount` return an error will reproduce the issue.

## Recommendations
1. Propagate the send error and abort the state transition, or explicitly roll back the deposit adjustments whenever the send fails.
2. Add integration tests for default-subaccount withdrawals under `SendEnabled=false` conditions.
3. Include an invariant that enforces module balances match the sum of stored deposits to catch similar regressions.

## Status
No remediation applied yet. Immediate patch is advised before feature releases or parameter changes that could flip `SendEnabled`.
