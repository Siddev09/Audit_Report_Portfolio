# Partial Withdrawal Wrongly Treated as Full Withdrawal (Position Deleted)

**Severity:** High

**Bug class:** Accounting / state logic error

## The Setup 

Imagine a fictional lock-vault called **StoneVault**, where users lock tokens for a fixed period and can later withdraw all or part of their locked amount.

```solidity
bool isFullRelease = (amount == position.amount);
// ... later ...
if (shares == userBalance) isFullRelease = true;
```

The contract decides "full vs. partial withdrawal" twice — once by comparing the requested amount, and again by comparing share balances.

## The Bug

After a lock expires, users are allowed to send some of their vault shares to a different wallet (a normal, intended feature). But if a user does that and *then* asks to withdraw only part of what's left, the contract's second check gets confused: it sees "your current balance equals the shares you're withdrawing" and wrongly concludes "you must be withdrawing everything."

Once it thinks that, it deletes the user's entire position record and clears the *whole* original locked amount from the books — even though the user only asked for, and only received, a partial amount. The rest of their money is now unaccounted for anywhere in the contract.

It's like a bank teller seeing you hand over your last $50 bill and assuming that means your entire account balance was $50, then closing your account and erasing the record of the other money that was still in it.

## Why It Matters

The user permanently loses the un-withdrawn portion of their deposit, with no function anywhere to restore a deleted position. Protocol-wide totals also become permanently wrong, which throws off any dashboard or system reading them.

## The Fix

Never let a share-balance comparison override the original, user-requested "full vs. partial" decision. Those are two different questions and should not share the same flag.
