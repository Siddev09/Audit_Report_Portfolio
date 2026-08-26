# Dead Guard in Schnorr Signature Verification (Sequencer Forgery)

**Severity:** Low
**Bug class:** Cryptographic verification / dead validation check

## The Setup 

Imagine a fictional L2 called **VelocityChain**. It has an off-chain "sequencer" that batches trades and submits them on-chain. The sequencer isn't supposed to be able to forge signatures — an on-chain `Verifier` contract checks every batch against a Schnorr signature before executing it.

```solidity
bytes32 sp = bytes32(Q - mulmod(uint256(s), uint256(px), Q));
require(sp != 0);
```

This line is meant to block a known "degenerate" math case that would let `ecrecover` accept a fake signature for any public key.

## The Bug

The check `require(sp != 0)` looks correct — but `sp` is calculated as `Q - something`, and that subtraction can *never* actually produce zero. The real dangerous value is `sp == Q`, not `sp == 0`. So the guard is checking for a condition that's mathematically impossible, while the actual bad input (`s = 0`) sails straight through untouched.

It's like installing a lock on a door that can never physically close — it looks like security, but it isn't doing anything.

## Why It Matters

Anyone who controls the sequencer's calling key can submit `s = 0` directly, no real signature needed, and the verifier will still say "valid" for any public key. That's the exact trust boundary the whole verification system exists to protect.

## The Fix

```solidity
require(sp != Q); // check the value that can actually occur
```

One-line fix — the guard just needed to check the right condition.
