# Reward-Tier Status Stolen From Other Users via Shared Staking Pool

**Severity:** High
**Bug class:** Access control / shared-state accounting

## The Setup 

Imagine a fictional loyalty-license NFT called **OriginPass**. Early holders get tagged as "founding members" (better rewards), and that tag is supposed to stick to the *person*, not move around.

```solidity
if (foundingBalance[from][id] > 0) {
    uint256 transferred = min(values[i], foundingBalance[from][id]);
    foundingBalance[from][id] -= transferred;
    foundingBalance[to][id] += transferred;
}
```

## The Bug

The "founding member" tag is meant to belong to one person forever. But when users stake their passes into a shared staking contract, the code treats the founding-member tag as if it were just another number that gets pooled together with everyone else's — instead of tracking whose tag belongs to whom.

So imagine two people, Alice (a real founding member) and Bob (a regular member), both stake their passes into the same shared pool. The pool doesn't remember "this portion is Alice's, this portion is Bob's" — it just sees one lump total. If Bob happens to withdraw *first*, the contract hands him some of the founding-member tag out of that shared lump, even though he never earned it. Alice, who staked it in the first place, ends up with less than she put in.

It's like two friends dropping their savings into one shared piggy bank, but the piggy bank doesn't track whose coins are whose — so whoever reaches in first gets to grab the "special" coins, regardless of who actually put them there.

## Why It Matters

Real founding members can permanently and irreversibly lose their status to other users just by both parties using the same staking pool — with no way to undo it after the fact, since nothing tracks the original ownership once it's pooled.

## The Fix

Track founding-member status per depositor *inside* the staking contract itself, instead of pooling it at the contract level — so withdrawal order can never reassign someone else's status.
