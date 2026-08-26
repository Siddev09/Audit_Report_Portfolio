# Staking Rewards Miscounted After Pool Sits Empty

**Severity:** duplicate

**Bug class:** Accounting / reward-distribution logic

## The Setup 

Imagine a fictional staking pool called **EmberStake**, where the reward-per-share value only updates when there's at least one staker present.

```solidity
if (totalWeights == 0) {
    return rewardPerShare; // frozen while pool is empty — correct in isolation
}
```

This freeze is intentional: if nobody is staking, nobody should be earning.

## The Bug

The problem shows up when the pool goes from empty back to having a staker again. The contract has a rule: "only update the reward clock if the reward-per-share number actually changed." But right when the pool is still empty, the reward-per-share number *can't* change (there's nothing to divide by), so the clock never gets nudged forward.

Then the first new staker arrives. As soon as they're in, the contract finally updates the clock — but it calculates elapsed time from the *old* frozen timestamp, which is from before the pool went empty. That means the entire idle period (with zero people staking) gets treated as if this one new staker had been earning rewards the whole time, alone.

It's like a taxi meter that pauses while the cab is empty, but when a new passenger gets in, it charges them for the entire time the cab sat empty at the curb — not just their actual ride.

## Why It Matters

The first staker back gets paid for a period they weren't even part of, and that payout comes straight out of the shared reward pool meant for everyone else going forward. Depending on how large the idle gap is, this can meaningfully drain the rewards budget.

## The Fix

When the pool transitions from empty back to active, reset the reward clock to the current time *before* calculating anything for the new staker — so idle time is never priced into anyone's payout.
