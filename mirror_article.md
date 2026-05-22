# Polymarket Match Engine — A Complete Trace

*Published May 22, 2026 · 12 min read*

I spent today taking apart Polymarket's matching engine.

Not reading docs. Not guessing. Tracing every function call from user signature to on-chain settlement, line by line, against the verified CTFExchange.sol on Polygon.

What I found: a surprisingly elegant design that the official documentation barely covers.

---

## 1. There is no AMM

This is the biggest misconception about Polymarket.

When you see `outcomePrices: ["0.17", "0.83"]` on the Gamma API, those numbers come from `/midpoint` — the CLOB's best bid/ask midpoint. Not from an AMM curve, not from a bonding curve, not from a liquidity pool.

Polymarket migrated from AMM to CLOB in mid-2023. The AMM is gone. The CLOB is the sole pricing layer.

I verified this by querying three CLOB endpoints against the same token and reconciling the results:

```
/book           → best_bid = 0.171, best_ask = 0.172
/midpoint       → mid = 0.1715
/price?side=buy → price = 0.171
/spread         → spread = 0.001

Gamma outcomePrices → YES=0.17, NO=0.83
               = /midpoint(四舍五入)
               = (0.171+0.172)/2 ≈ 0.17
```

The numbers match perfectly.

## 2. The unified order book with mirror translation

Here's something the docs don't tell you.

When you query `/book?token_id=YES_token`, you get 168 orders. When you query `/book?token_id=NO_token`, you get **zero**.

This isn't a bug. Polymarket maintains a **single unified order book** for each market. All NO-side orders are internally mirrored into YES-side equivalents:

```
BUY NO @ 0.83  →  mirrored as SELL YES @ 0.17
SELL NO @ 0.15 →  mirrored as BUY YES @ 0.85
```

This mirroring is why Polymarket achieves a 0.001 spread on a $23M-volume market. If YES and NO books were separate liquidity pools, the spread would be wider because orders on each side wouldn't see each other. The unified book aggregates everything onto one price ladder.

This is also why `calculate_sell_market_price` in py-clob-client iterates `reversed(positions)` — the /book endpoint returns bids in ascending order and asks in descending order. The best prices are at the **end** of each array, not the beginning. (I got this wrong twice before checking the source.)

## 3. The EIP-712 Order: exactly 12 fields

The canonical Order structure is defined in `ORDER_TYPEHASH` on the verified CTFExchange contract. The typehash constant is:

```solidity
bytes32 constant ORDER_TYPEHASH = keccak256(
    "Order(uint256 salt,address maker,address signer,address taker,uint256 tokenId,uint256 makerAmount,uint256 takerAmount,uint256 expiration,uint256 nonce,uint256 feeRateBps,uint8 side,uint8 signatureType)"
);
```

| Field | Type | Purpose |
|-------|------|---------|
| salt | uint256 | Random salt, prevents replay |
| maker | address | Address holding the funds |
| signer | address | Address that signs the order |
| taker | address | 0x0 = public order |
| tokenId | uint256 | CTF ERC1155 token being traded |
| makerAmount | uint256 | Maximum tokens to sell |
| takerAmount | uint256 | Minimum tokens to receive |
| expiration | uint256 | Unix timestamp (0 = never expires) |
| nonce | uint256 | For on-chain cancellation |
| feeRateBps | uint256 | Fee in basis points |
| side | uint8 | 0 = BUY, 1 = SELL |
| signatureType | uint8 | 0 = EOA, 1 = POLY_PROXY, 2 = POLY_GNOSIS_SAFE |

No `outcomeIndex` field. No `orderId` field. The outcome is determined by `tokenId` alone.

---

*The remaining 60% of this article covers the matching engine decision logic, the three settlement branches, a complete on-chain trace of _matchOrders with asset balance tracking, and five specific risk areas in the current architecture.*

---

# Gated Content

*This section is available to token holders.*

*(NFT gate would go here on Mirror)*

---

### The matching engine: _isCrossing and _deriveMatchType

*(technical content below — paywall placeholder)*

The matching engine uses two functions from Trading.sol to decide how to settle each match:

**`_isCrossing` (CalculatorHelper.sol L74-88)** checks price compatibility:

```
BUY × BUY:   priceA + priceB >= 1      → MINT condition
BUY × SELL:  priceA >= priceB           → COMPLEMENTARY condition
SELL × BUY:  priceB >= priceA           → COMPLEMENTARY condition
SELL × SELL: priceA + priceB <= 1      → MERGE condition
```

**`_deriveMatchType` (Trading.sol L230-234)** dispatches to one of three match types:

```solidity
if (taker.side == BUY && maker.side == BUY) return MINT;
if (taker.side == SELL && maker.side == SELL) return MERGE;
return COMPLEMENTARY;
```

The key constraint: COMPLEMENTARY matches **must have the same tokenId**. The contract explicitly checks this in `_validateTakerAndMaker`:

```solidity
if (matchType == MatchType.COMPLEMENTARY) {
    if (takerOrder.tokenId != makerOrder.tokenId) revert MismatchedTokenIds();
}
```

This means a BUY YES taker cannot directly match a SELL NO maker. That combination would require a MINT path (with another BUY NO maker) or a MERGE path (with another SELL YES maker) as an intermediary. Cross-token matching is not part of the COMPLEMENTARY path.

### The three settlement branches

**COMPLEMENTARY (buy vs sell, same tokenId):**

The simplest path. `_executeMatchCall` returns immediately — it's a nop. The tokens already exist. The Exchange simply orchestrates the transfer:

```
Maker → Exchange: CTF token
Exchange → Maker: USDC (proceeds minus fees)
```

**MINT (both buy, complementary tokens):**

When two buyers want opposite outcomes, the Exchange pools their USDC and calls `CTF.splitPosition()`:

```solidity
// Taker BUY YES @ 0.30, Maker BUY NO @ 0.70
// Exchange pools: 0.30 + 0.70 = 1.00 USDC
CTF.splitPosition(USDC, conditionId, [1,2], 1_000_000)
// → 1 YES + 1 NO created
// → Taker receives YES, Maker receives NO
```

The critical observation: during MINT execution, the Exchange's USDC balance drops to zero (consumed by splitPosition), and YES+NO emerge from nothing. The asset **changes form** inside the transaction.

**MERGE (both sell, complementary tokens):**

The reverse of MINT. Two sellers provide complementary tokens, the Exchange calls `CTF.mergePositions()`:

```solidity
// Taker SELL YES, Maker SELL NO
CTF.mergePositions(USDC, conditionId, [1,2], 1)
// → 1 YES + 1 NO destroyed
// → 1.00 USDC returned
// → Distributed by price ratio
```

### Complete _matchOrders trace

The following trace follows a hypothetical mixed batch through `_matchOrders()` (Trading.sol L105-147):

**Scenario:** Taker BUY YES, 3 makers:
- Maker A: SELL YES @ 0.172, 15 shares → COMPLEMENTARY
- Maker B: BUY NO @ 0.83, 20 shares → MINT
- Maker C: SELL YES @ 0.50, 10 shares → COMPLEMENTARY

**Exchange balance tracking:**

```
Step 0: Exchange = {USDC: 0, YES: 0, NO: 0}

Step 1 — Taker deposits: _transfer(taker→Exchange, USDC, 45)
         Exchange = {USDC: 45, YES: 0, NO: 0}

Step 2 — Maker A COMPLEMENTARY:
  makerA sends YES→Exchange    : {USDC: 45, YES: 15, NO: 0}
  Exchange pays USDC→makerA    : {USDC: 42.42, YES: 15, NO: 0}

Step 3 — Maker B MINT:
  makerB sends USDC→Exchange   : {USDC: 62.42, YES: 15, NO: 0}
  splitPosition(20 USDC)       : {USDC: 42.42, YES: 35, NO: 20}
  Exchange pays NO→makerB      : {USDC: 42.42, YES: 35, NO: 0}

Step 4 — Maker C COMPLEMENTARY:
  makerC sends YES→Exchange    : {USDC: 42.42, YES: 45, NO: 0}
  Exchange pays USDC→makerC    : {USDC: 37.42, YES: 45, NO: 0}

Step 5 — Taker receives YES:
  Exchange→taker: 45 YES       : {USDC: 37.42, YES: 0, NO: 0}

Step 6 — Refund leftover USDC:
  Exchange→taker: 37.42 USDC   : {USDC: 0, YES: 0, NO: 0}
```

**Taker net result:** Spent 7.58 USDC, received 45 YES. Effective price: 0.168 YES/USDC.

### Risk areas identified

1. **MINT USDC consumption:** During MINT, all pooled USDC is consumed by splitPosition. If the sum of taker + maker USDC doesn't exactly equal the mint amount, the transaction reverts. No partial mints.

2. **COMPLEMENTARY tokenId constraint:** BUY YES × SELL NO is invalid. Integrators who don't check tokenId compatibility will get MismatchedTokenIds reverts. Same-side matching requires complementary tokens (YES↔NO), not arbitrary token pairs.

3. **surplus assignment:** _updateTakingWithSurplus gives all excess to the taker. Makers who provide tokens for MERGE don't receive any surplus from rounding. This creates an asymmetry — the taker is structurally favored in the batch design.

4. **Mixed batch atomicity:** All N makers + taker settle in one transaction. A revert from any single maker invalidates the entire batch. The operator must ensure all included orders are valid (nonce, fill status, balance) before constructing the batch.

5. **Exchange as escrow:** Between maker fills, the Exchange contract holds all intermediate assets. Any reentrancy vulnerability or unexpected state change during _executeMatchCall could compromise the batch. (No known vulnerability — the contract uses standard ERC1155/ERC20 calls with no external callbacks — but the centralization of assets during execution is worth noting.)

---

### What's next

I built three interactive diagrams covering the full lifecycle, matching decision, and this on-chain trace. Each node is clickable and shows the exact Solidity source code.

**GitHub:** [polymarket_architecture](https://github.com/polymarket_architecture)

If you're building on Polymarket's CLOB or auditing the CTF Exchange contract, these should save you a few hours of reading raw Solidity.

*For technical questions, find me on X or the Polymarket Discord.*
