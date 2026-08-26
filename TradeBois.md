# Auction Can Be Permanently Bricked by a Blacklisted Bidder

**Severity:** High in DualDefence Contest  
**Bug class:** Denial of service / unsafe external call handling

## The Setup 

Imagine a fictional NFT auction house called **CrestAuction**. Whenever someone places a new bid, the contract immediately refunds the previous highest bidder before accepting the new one.

```solidity
paymentToken.safeTransferFrom(msg.sender, address(this), amount);

if (_auction.bidder != address(0)) {
    paymentToken.safeTransfer(_auction.bidder, _auction.amount);
    // if this transfer fails, the WHOLE transaction reverts
}
```

## The Bug

The payment token used for bidding has a "blacklist" feature (common in tokens like USDC) — if an address gets blacklisted, it can no longer send or receive that token.

The auction contract refunds the previous bidder automatically, in the same transaction as the new bid. If that previous bidder is ever blacklisted, the refund transfer fails, and because the contract does no error handling around it, the *entire* new bid transaction reverts too — not just the refund.

So an attacker just needs to place the very first bid, then get their own address blacklisted (deliberately, e.g. by triggering a compliance flag). After that, nobody can outbid them: every attempt to bid higher tries to refund the attacker first, that refund always fails, and the whole transaction always reverts. The auction is frozen forever with the attacker as the "winner" at the lowest possible price — and since blacklists only block token transfers, not NFT transfers, the attacker can still walk away with the NFT.

It's like a raffle where, to accept a new ticket, the box first has to mail a refund check to the previous ticket holder — and if that person's mailbox has been condemned, the post office refuses to process *any* further mail, jamming the entire raffle shut with them still holding the only valid ticket.

## Why It Matters

- Total denial of service on the auction — no legitimate bidder can ever outbid the attacker again.
- Attacker wins high-value NFTs for the minimum possible bid.
- Requires minimal capital, no special access, and is trivially repeatable across every auction the contract runs.

## The Fix

Never let a failed refund block the new bid from succeeding. Common patterns:
- Use a **pull-based refund** system instead (credit the previous bidder's balance in a mapping, let them withdraw it separately) so a blocked recipient never affects anyone else.
- If keeping push-based refunds, wrap the refund transfer in a try/catch so a failure only forfeits that bidder's refund, without reverting the new bid.
