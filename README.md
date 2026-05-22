# Polymarket Match Engine

Three interactive diagrams tracing Polymarket's order flow from EIP-712 signature to on-chain CTF Exchange settlement.

- **`order_lifecycle.html`** — Full lifecycle: market query → parameter validation → EIP-712 signing → ClobAuth → HMAC → POST /order → operator validation → CLOB matching → settlement. 19 nodes. HTTP errors (4xx) and WS events are drawn as separate notification channels.
- **`matching_subdiagram.html`** — Matching decision detail: unified YES book scan with NO mirror translation, `_isCrossing` price compatibility, `_deriveMatchType` dispatch into three branches (COMPLEMENTARY / MINT / MERGE). 10 nodes.
- **`matchorders_trace.html`** — Step-by-step trace of `_matchOrders` against the verified Solidity source (Trading.sol L105-268, CalculatorHelper.sol L74-88). Exchange asset balance tracked per step across a mixed batch of all three match types.

Built from the `ctf-exchange` verified contract on Polygon (0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E) and the py-clob-client SDK. Gamma API `outcomePrices` are derived from `/midpoint`; no AMM component was found in the current pricing path.

Scope: user signature → on-chain settlement. Out of scope: UMA resolution, Polygon consensus.

## License

MIT
