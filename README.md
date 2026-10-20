# Mobile Money Trust Account Reconciliation

How a **mobile money platform** proves every day that the e-money in customer, agent and merchant wallets is backed by real money in the bank. Covers what counts as e-money liability, the bank side of the match, the timing items that make the two differ, and what to do when there is a genuine shortfall. Written for finance, treasury and operations teams evaluating [EVD System](https://evdsystem.com/) by [MoboGage](https://evdsystem.com/about-mobogage/).

---

## What trust account reconciliation is

**Trust account reconciliation** is the daily check that the total e-money outstanding on a mobile money platform equals the funds held for customers in designated bank accounts. Depending on the country these accounts are called trust, escrow, safeguarding or pooled accounts. The idea is the same everywhere: e-money is a claim on real money, and that money must sit somewhere ring-fenced from the operator's own cash.

The e-money side comes from the wallet ledger. The bank side comes from bank statements. When the two do not agree after timing items are explained, the difference is a shortfall or an excess, and both need action.

The exact rules (which banks qualify, how fast a shortfall must be fixed, what happens to interest earned) are set by the central bank or e-money regulator in each market. This repo describes the mechanics, not any one country's regulation.

## Issuing and redeeming e-money

It helps to be strict about when e-money is created and destroyed.

**Issuance.** E-money should be created only after money reaches the trust account. A super-agent deposits cash at the bank, the deposit shows on the statement or comes through the bank's API, and only then does the platform credit the super-agent's float wallet. Treasury teams sometimes call the controlled wallet this comes from the issuance or mint account.

**Redemption.** E-money is destroyed when it leaves the system for real money: a wallet-to-bank payout, an agent selling float back to the operator for a bank transfer, or the operator sweeping its earned fees out to its operating account.

**Movement inside the system.** P2P transfers, cash-in and cash-out at agents, merchant payments and bill payments to billers who hold wallets all move e-money between wallets. They do not change the total, so they do not touch the trust account.

If anyone can credit a wallet without a matching bank receipt, through a manual adjustment screen or a test script left enabled in production, the reconciliation will catch it. It is far better that the controls prevent it.

## What makes up the e-money liability

Sum every wallet balance the operator owes to someone. That usually includes:

| Ledger group | Notes |
|-------------|-------|
| Customer wallets | All KYC tiers, including dormant and frozen wallets |
| Agent and super-agent float | Float is e-money even though agents hold it for business |
| Merchant and biller wallets | Includes unsettled merchant collections |
| Commission wallets | Agent commission already credited is owed to the agent |
| Suspense and holding wallets | Failed transfers, unclaimed remittances, items pending investigation |
| Operator fee wallet | Fees earned but not yet swept. Owed to the operator, still backed by trust funds until swept |

Wallets with a negative balance should not exist. If the ledger allows them, report them separately instead of netting them off, because a negative wallet hides a loss inside the total.

## The reconciliation

```
Trust bank balances (closing, all banks, per statement)
  + payouts already posted in the ledger, not yet debited by the bank
  - bank receipts not yet issued as e-money (unloaded deposits)
  + bank charges taken from the trust account, not yet replaced
  - interest credited, if it must be moved out per local rules
  +/- interoperability or partner settlement in transit
  = Expected e-money liability
      compared with
  E-money liability per ledger at the same cut-off
  = Difference (should be zero)
```

**Same cut-off on both sides.** If the ledger snapshot is taken at 23:59:59 local time, the bank side must be adjusted to the same moment. Many breaks that look scary are just a ledger snapshot taken at midnight against a bank statement that closes at 18:00.

**Every timing item named.** "Deposits in transit 3,250.00" is not enough. Each timing item should have a reference so it can be checked off when it clears. Items that do not clear in a few days are no longer timing items.

**Several banks.** Larger operators split trust funds across banks to limit concentration risk. Reconcile per bank first, then in total. A total that matches can hide one bank that is short and another that is over.

## Common breaks and where they come from

**Unloaded deposits.** A super-agent paid in cash at the bank but the reference was wrong, so the platform never issued the float. The bank side is higher than the ledger. Find the depositor, issue the e-money, and fix the reference format on deposit slips.

**Double issuance.** The bank API sent the same credit notification twice and the platform credited the float twice. The ledger is higher than the bank. Every bank receipt needs a unique reference and the issuance step must reject a reference it has already used.

**Bank charges.** The bank debits fees from the trust account. Unless the operator replaces those funds from its own money, the trust account slowly falls below the liability.

**Failed payouts.** A wallet-to-bank payout was debited from the wallet, the bank rejected it, and the money came back to the trust account, but the wallet was never credited. Customers notice this one quickly.

**Fee sweeps done early.** Treasury sweeps fees out of the trust account before the fee wallet was debited in the ledger, or sweeps more than the fee wallet holds.

**Manual adjustments.** Goodwill credits and corrections posted by support without a matching funding entry.

## Shortfall handling

A shortfall is a regulatory event in many markets. A workable internal policy:

1. Raise an alert as soon as the unexplained difference crosses a small tolerance.
2. Treasury tops up the trust account from operator funds the same day, before the cause is fully known.
3. Operations investigates and classifies the cause.
4. If the cause is recovered later (for example a duplicate credit clawed back from a wallet), the operator moves its top-up back out with a documented approval.

An **excess** is less urgent but still matters. It usually means customer money arrived and was not issued, and someone is waiting for their float.

## Worked example (figures are illustrative)

All figures in this section are **illustrative**.

Cut-off 23:59:59 local time. Two trust banks.

| Bank side | Amount |
|-----------|--------|
| Bank A closing balance | 1,840,000.00 |
| Bank B closing balance | 612,500.00 |
| Payouts posted in ledger, not yet debited by bank | +4,200.00 |
| Deposits received, not yet issued as e-money | -15,000.00 |
| Bank charges not yet replaced | +350.00 |
| **Expected e-money liability** | **2,442,050.00** |

| Ledger side | Amount |
|-------------|--------|
| Customer wallets | 1,205,300.00 |
| Agent and super-agent float | 1,061,900.00 |
| Merchant and biller wallets | 141,200.00 |
| Commission wallets | 18,400.00 |
| Suspense | 6,750.00 |
| Operator fee wallet | 7,000.00 |
| **E-money liability per ledger** | **2,440,550.00** |

Difference: 2,442,050.00 minus 2,440,550.00 = **1,500.00 excess** on the bank side.

Investigation finds a 1,500.00 wallet-to-bank payout that the bank returned that afternoon. The trust account got the money back, the customer's wallet did not. The fix is to credit the wallet with a reference to the return, after which the difference is zero. The 15,000.00 unloaded deposit is a separate open item that must be traced to a depositor.

## Fields worth keeping for each daily run

| Field | Why |
|-------|-----|
| `recon_date`, `cutoff_at` | Both sides measured at the same moment |
| `bank_id`, `statement_ref`, `closing_balance` | One row per trust bank |
| `liability_by_group` | Lets you see which ledger group moved |
| `timing_items[]` with `ref`, `age_days` | Ageing shows when a timing item has become a real break |
| `difference`, `tolerance` | Result and the threshold used |
| `prepared_by`, `approved_by` | Maker-checker on the sign-off |
| `topup_ref` | Link to any shortfall funding |

The daily runbook is in [docs/daily-trust-recon-runbook.md](./docs/daily-trust-recon-runbook.md).

---

## FAQ

**What is trust account reconciliation in mobile money?**
It is the daily check that the total e-money in all wallets on a mobile money platform equals the funds held for those wallet holders in ring-fenced trust, escrow or safeguarding bank accounts.

**Do P2P transfers and agent cash-in affect the trust account?**
No. They move e-money between wallets inside the system. Only issuance against bank receipts and redemption out to bank accounts change the total.

**Is agent float part of the e-money liability?**
Yes. Agent and super-agent float is e-money owed by the operator, so it must be backed by trust funds like customer balances.

**What causes a trust account shortfall?**
Common causes are double issuance from duplicate bank notifications, bank charges taken from the trust account, early fee sweeps and manual wallet credits posted without funding.

**What should an operator do when the trust account is short?**
Top it up from operator funds the same day, then investigate. Local regulation usually sets the deadline and reporting duty.

**Does EVD System cover mobile money as well as vouchers?**
MoboGage's broader stack includes mobile money and digital wallet alongside the EVD System voucher platform.

---

## Glossary

| Term | Meaning |
|------|---------|
| E-money liability | Total of all wallet balances the operator owes to holders |
| Trust account | Ring-fenced bank account holding funds that back e-money. Also called escrow, safeguarding or pooled account |
| Issuance | Creating e-money after funds reach the trust account |
| Redemption | Destroying e-money when value leaves for a bank account |
| Timing item | A difference expected to clear, such as a payout in transit |
| Unloaded deposit | Money received in the trust account but not yet issued to a wallet |

## Further reading

- [Mobile money platform with merchant payment](https://evdsystem.com/mobile-money-platform-with-merchant-payment/) on evdsystem.com
- [MoboGage case studies](https://evdsystem.com/case-studies/) on evdsystem.com
- Related repo: [prepaid-ewallet-ledger](https://github.com/iwtglobal/prepaid-ewallet-ledger)

## License

MIT. See [LICENSE](./LICENSE). Educational material only, not legal, accounting or regulatory advice.
