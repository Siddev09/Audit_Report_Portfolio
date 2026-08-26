# 1. Anyone Could Cancel Someone Else's Sell Order

**Severity:** Critical

**Bug class:** Access control (missing authorization)

## The Setup 

Imagine a fictional on-chain order book called **DriftMarket**, where users place sell orders and can cancel them later.

```solidity
function _cancelSellOrder(uint256 option, uint256 price, uint256 orderID, address caller) private {
    address sellerAddress = getMaker(nodeIndex);

    // MISSING: if (caller != sellerAddress) revert();

    remove(nodeIndex);
    userActiveSellOrders[caller]--;
    userVotesInEscrow[option][caller] -= orderAmount;
}
```

## The Bug

The function to cancel a *buy* order correctly checks "are you actually the person who placed this order?" before doing anything. The function to cancel a *sell* order — doing basically the same job — simply forgot that check.

So any user could look up someone else's open sell order ID and cancel it themselves. Worse, the accounting update at the end (reducing "active orders" and "escrowed funds") gets applied to the *attacker's* account instead of the victim's, because the code trusts whoever called the function, not who actually owned the order.

It's like a coat-check where anyone holding any ticket number can hand in *any* coat tag and walk off with someone else's coat — and the counter still marks it against their own ticket instead of the coat's real owner.

## Why It Matters

An attacker can silently cancel any other user's sell orders at will. The victim's order disappears from the book without their consent, but their internal accounting (active order count, escrowed funds) never gets cleared — leaving them stuck and unable to fix it themselves.

## The Fix

```solidity
if (caller != sellerAddress) revert();
```

Add the same ownership check that already exists on the buy-order cancellation path.



---


# 2.  Pool Could Be Closed Before Its Promised End Time

**Severity:** Low

**Bug class:** Trust assumption / design gap

## The Setup 

Imagine a fictional prediction pool called **TidePool**, created with a fixed `endTime` shown to users so they know how long they have to enter or adjust their positions.

## The Bug

The function that closes the pool never actually checks whether `endTime` has been reached. It just lets the pool owner call it whenever they want. So even though users see "this pool runs until X," the owner can shut it down well before that, cutting off anyone who was still planning to act.

It's like a store posting "open until 9pm" on the door, but the owner can lock up at 6pm any day they feel like it — nothing stops them, and nothing warns customers.

## Why It Matters

Users who trusted the advertised time window lose the ability to enter or adjust positions early, and the pool owner effectively gets discretion over outcomes by choosing when to cut things off.

## The Fix

Either enforce `block.timestamp >= endTime` before allowing the pool to close, or clearly document that the end time is just an upper bound and early closure is expected — so users aren't relying on a guarantee that doesn't actually exist.
