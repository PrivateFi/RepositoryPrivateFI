# Integration

A guide for frontend and indexer developers. Function and event names match the ABIs in
[`abi/`](./abi).

## Contract addresses

Only the three shared contracts need to be configured: **LaunchFactory**, **Hook** and **Locker**,
plus the canonical Uniswap v4 **PoolManager**. Every Token and Rewards address can be found
through the Factory.

> Official deployment addresses are published on [privatefi.app](https://privatefi.app).
> Always check an address against the Factory registry before trusting it.

## 1. Discover launches through the Factory

**From events (recommended for indexers).** Index `PrivateFiLaunchFactory.TokenLaunched`:

```text
TokenLaunched(
  uint256 indexed launchId, address indexed token, address indexed creator,
  address rewards, bytes32 poolId, string name, string symbol, string metadataURI,
  uint256 supply, uint16 buyFeeBps, uint16 sellFeeBps, uint256 openMarketCapWei, int24 openTick)
```

One event carries everything you need to list a launch. Add `PrivateFiHook.PoolListed` (pool
config and tick spacing) and `PrivateFiLocker.LiquidityLocked` (the locked range) to confirm the
pool and its position.

**From views (for frontends without an indexer).**

- `launchCount()` gives the number of launches. IDs are 1-based and sequential.
- `getLaunches(fromId, count)` returns a page of `LaunchInfo` (at most `MAX_PAGE` = 100 per call).
- `getLaunch(launchId)` / `getLaunchByToken(token)` return a single launch.
- `launchCountOf(creator)` / `launchIdsOf(creator, offset, count)` return launches by creator.

`LaunchInfo` = `{ token, rewards, creator, poolId, buyFeeBps, sellFeeBps, openTick, launchedAt, openMarketCapWei }`.

## 2. Identify tokens by address / launch ID

- The **token address** is the canonical identifier. `launchIdOf(token)` returns its launch ID
  (`0` = not a PrivateFi launch). `isLaunched(token)` returns the same as a bool.
- The **launch ID** is a stable, sequential index for URLs and sorting.
- `poolKeyOf(token)` returns the full v4 `PoolKey`. `PrivateFiHook.poolIdOf(token)` returns the
  `PoolId`.
- Name and symbol are chosen by the creator and **not unique**. Never identify a token by
  name or symbol alone.

## 3. Get market data from Uniswap v4 pool activity

Every PrivateFi pool has this shape:

| Field | Value |
|---|---|
| `currency0` | native ETH (`address(0)`) |
| `currency1` | the token |
| `fee` | `0x800000` (dynamic-fee flag; LP fee is always 0) |
| `tickSpacing` | `60` |
| `hooks` | PrivateFiHook |

- **Price.** Read `slot0` for the `PoolId` through the PoolManager (for example with a
  StateView-style lens). `price (tokens per ETH) = (sqrtPriceX96 / 2^96)^2`, adjusted for decimals
  (both 18). The token price in ETH is the reciprocal.
- **Market cap.** Token price × `totalSupply()`. The supply is fixed, so no supply changes need
  tracking.
- **Liquidity.** The whole supply is one permanent position owned by the Locker.
  `PrivateFiLocker.positionOf(poolId)` returns its range and liquidity.
- **Fees.** `PrivateFiHook.feesOf(poolId)` returns `(buyFeeBps, sellFeeBps)`. They never change, so
  you can cache them.

## 4. Build charts from swaps

Index the PoolManager's `Swap` event filtered by PrivateFi `PoolId`s. Join it with
`PrivateFiHook.SwapFeeRouted` from the same transaction:

```text
SwapFeeRouted(bytes32 indexed poolId, bool indexed isBuy, uint256 ethAmount, uint256 feeBps, uint256 fee)
```

- Each `Swap` gives `sqrtPriceX96` and `tick` after the swap, which is enough for OHLC candles.
- `Swap.amount0` / `amount1` describe the **pool** swap. The Hook fee is taken separately, as a
  hook delta, and is **not** included:
  - Exact-input buy: the fee is removed before the pool swap. Gross ETH paid = pool ETH + `fee`.
  - Exact-output buy: gross ETH paid = pool ETH + `fee`.
  - Exact-input sell: net ETH received = pool ETH − `fee`.
- `SwapFeeRouted.isBuy` gives the side, and `fee` is the exact ETH sent to Rewards.
- Check the sign convention of `amount0` / `amount1` against a known buy on the deployed PoolManager.
- `Swap.sender` is the router that called the PoolManager, not the end user. Use the transaction
  sender or the token `Transfer` events to attribute trades.

### Current swap-mode limitations

- **Exact-output sells are not supported.** A token → ETH swap with a positive `amountSpecified`
  reverts with `ExactOutputSellDisabled`. Routers and quoters must use exact-input sells.
- Exact-input buys, exact-output buys and exact-input sells are all supported.
- Only ETH pairs exist, with exactly one pool per token. There are no token/token pools on the hook.

## 5. Resolve Rewards through the Factory

- `PrivateFiLaunchFactory.rewardsOf(token)` returns the token's Rewards vault. It is also in
  `LaunchInfo.rewards` and `TokenLaunched.rewards`.
- `PrivateFiToken.vault()` should return the same address. Treat the Factory registry as
  authoritative.

## 6. Show pending rewards

- **Many tokens at once:** `PrivateFiLaunchFactory.pendingRewards(tokens[], account)` returns
  `(uint256[] amounts, uint256 total)`. Unknown tokens return 0 instead of reverting.
- **Single token:** `PrivateFiRewards.pendingEth(account)`.
- **Idle status:** `PrivateFiRewards.recoverableAt(account)` (0 if the clock never started) and
  `isRecoverable(account)` → `(eligible, amount)`. Show a countdown so holders know when they must
  claim before their rewards can be recovered.
- **Vault stats:** `lifetimeDeposits`, `lifetimeWithdrawn`, `lifetimeRecovered`, `buybackReserve`,
  `outstandingEth`.

To build a user's token list, collect the tokens they have held (from `Transfer` events) and query
`pendingRewards` for them.

## 7. Use Claim All

```text
PrivateFiLaunchFactory.claimAll(address[] tokens) → uint256 total
```

- It is the **only** claim path. Rewards contracts have no direct claim function for users.
- ETH is always paid to `msg.sender`. There is no recipient parameter.
- Pass only launched tokens: an unlisted token reverts the whole call with `UnknownToken(token)`.
- Tokens with nothing owed are skipped, but they still cost gas. Filter with `pendingRewards`
  first.
- The caller's gas limit bounds the list length. For very long lists, split them into several
  calls.
- A claim restarts the caller's 24h idle clock in each vault that paid out.
- If the caller is a contract that rejects ETH, the call reverts with `PayoutReverted`.

## 8. Index reward and recovery events

| Contract | Event | Use |
|---|---|---|
| Factory | `TokenLaunched` | New launch |
| Factory | `FirstBuy(launchId, buyer, ethIn, tokensOut, ethRefunded)` | Creator's first buy |
| Factory | `ClaimAll(account, tokenCount, totalClaimed)` | Claim summary per call |
| Factory | `IdleRecovered(token, caller, accountCount, totalRecovered)` | Recovery summary per call |
| Factory | `LaunchPausedSet(paused)` | Show launch availability |
| Hook | `PoolListed` | Pool config |
| Hook | `SwapFeeRouted` | Fee per swap, for volume and fee analytics |
| Locker | `LiquidityLocked` | Locked position details |
| Token | `Transfer` | Holder balances |
| Token | `Bootstrapped`, `RewardExempted` | Token initialization and excluded addresses |
| Rewards | `FeesDeposited(amount, distributed, accEthPerWeight, eligibleSupply)` | Fee inflow per vault |
| Rewards | `RewardsWithdrawn(account, amount, caller)` | Per-account claims |
| Rewards | `IdleRewardsRecovered(account, amount, expiredAt)` | Per-account recovery |
| Rewards | `BuybackReserveGrew(amount, newReserve)` | Reserve growth |
| Rewards | `BuybackReserveReleased(amount)` | Reserve paid to the buyback wallet |

The buyback wallet's own holder claims do not emit `RewardsWithdrawn`, and
`BuybackReserveReleased` carries no address. This is deliberate. See
[SECURITY.md](./SECURITY.md#privacy-clarification).

## 9. Launching from a frontend

- Preview the opening price with `openPriceFor(supply, openMarketCapWei)` (pure). The realised
  opening market cap is the target or up to one tick-spacing step (~0.6%) above it.
- Validate inputs against the Factory constants before sending: `MIN/MAX_LAUNCH_SUPPLY`,
  `MAX_FEE_BPS`, `MIN/MAX_OPEN_TICK`, `MAX_NAME_LENGTH`, `MAX_SYMBOL_LENGTH`,
  `MAX_METADATA_URI_LENGTH`.
- Check `launchPaused()` before submitting.
- Any `msg.value` sent with `launch` becomes the creator's first buy.
