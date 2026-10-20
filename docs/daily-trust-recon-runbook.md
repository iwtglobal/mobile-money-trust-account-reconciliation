# Daily Trust Account Reconciliation Runbook

Steps for finance and operations to run the daily e-money versus trust account match on a mobile money platform. Replace every figure with your own policy values.

## Before 10:00 (illustrative deadline)

1. Pull closing statements for every trust bank for the previous business day. Use the bank API where available and keep the raw file.
2. Pull the ledger liability snapshot at the agreed cut-off, grouped by customer, agent, merchant, commission, suspense and fee wallets.
3. Confirm both snapshots use the same cut-off time. If the bank closes earlier, list the ledger movements between the two times as timing items.

## Build the match

4. Add payouts posted in the ledger but not yet on the bank statement. Each needs a reference.
5. Subtract receipts on the statement not yet issued as e-money.
6. Add bank charges not yet replaced by the operator.
7. Handle interest per local rules.
8. Reconcile each bank on its own, then the total.

## Check the open items

- [ ] Timing items older than the ageing limit (**illustrative**: 3 business days) moved to the break list
- [ ] Every bank receipt reference used once only in issuance
- [ ] Returned payouts matched to a wallet re-credit
- [ ] Manual adjustments from the day listed with their funding entries
- [ ] No wallet with a negative balance
- [ ] Fee wallet debited before or with any fee sweep

## If there is a shortfall

- [ ] Alert treasury and the head of finance once the difference passes tolerance (**illustrative**: 100.00)
- [ ] Same-day top-up from operator funds, with `topup_ref` recorded
- [ ] Root cause recorded and controls fixed
- [ ] Regulatory reporting done where required

## If there is an excess

- [ ] Trace each unloaded deposit to a depositor and issue the e-money
- [ ] Unidentified receipts kept in a named suspense item, not left in the gap

## Sign-off

- [ ] Prepared and approved by different people
- [ ] Recon pack (statements, ledger snapshot, item list) archived for audit
