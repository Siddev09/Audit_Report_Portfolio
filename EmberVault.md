# Withdrawal Fee Bypassed by Moving Shares to a Fresh Wallet

**Severity:** Duplicate
**Bug class:** Access control / economic logic bypass

## The Setup 

Imagine a fictional yield vault called **HarborVault**. To discourage people from depositing and immediately withdrawing to farm short-term rewards, it charges a fee if you withdraw too soon after depositing.

```solidity
uint256 lastDeposit = _lastDepositTimestamp[vault][owner_];
if (lastDeposit > 0 && lastDeposit + fee.timeBasedFeeThreshold > currentTime) {
    timeBasedFeeCharged = FixedPointMath.mul(withdrawAmount, tbFeePercent);
}
```

The fee only applies if the *withdrawing address* has a recorded deposit timestamp.

## The Bug

The vault only writes down "this address deposited at this time" — it never updates that record when shares are simply transferred to someone else. Since the vault shares are ordinary, freely-transferable tokens, a user can deposit, immediately send the shares to a brand-new wallet that has never deposited anything, and then withdraw from that new wallet.

That new wallet has no deposit timestamp at all, so the fee check ("did you deposit recently?") sees nothing and charges nothing — even though the actual deposit happened moments ago.

It's like a "no returns within 30 days" store policy that only checks the name on the *original* receipt — so you just hand the item to a friend, and they return it under their own name with no receipt at all.

## Why It Matters

Anyone aware of this can completely dodge the intended short-term-flip fee, indefinitely, just by routing funds through one extra wallet each time. That's lost fee revenue and a fully defeated economic control.

## The Fix

Tie the deposit-history check to the *shares themselves* moving through transfers, not just to whichever address originally called `deposit()`.
