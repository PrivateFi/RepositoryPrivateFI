# Testing

A checklist of the behaviours a PrivateFi deployment must be verified against. The private test
suite is written in Foundry and runs against a locally deployed Uniswap v4 PoolManager.

## Deployment / wiring

- [ ] Implementations deployed with the predicted Factory as `BOOTSTRAPPER`. The Factory
      constructor rejects mismatched implementations.
- [ ] The Hook's CREATE2 address carries exactly the required permission bits. The constructor
      rejects any other address.
- [ ] `handOffRegistrar` works once only, and only from the initial registrar.
- [ ] `wire` works once only, only from the deployer, and only after the handoff with a matching
      PoolManager and Locker.
- [ ] `launch` reverts before wiring (`NotWired`).
- [ ] No pool can be registered on the Hook by anyone other than the Factory after the handoff.

## Permissionless launches

- [ ] The genesis launch is owner-only (`GenesisOwnerOnly`). Afterwards any account can launch.
- [ ] Bounds are enforced: supply, fees ≤ `MAX_FEE_BPS`, opening tick range, name, symbol and URI
      lengths.
- [ ] `setLaunchPaused` blocks only new launches. Trading, fees, claims and recovery continue.
- [ ] Invalid input always reverts cleanly and leaves no partial state.

## Atomic launch flow

- [ ] One transaction performs: clone Rewards → clone Token → initialize → register pool →
      initialize v4 pool → lock liquidity → optional first buy.
- [ ] Nobody can initialize the pool outside `registerPool` (`beforeInitialize` reverts).
- [ ] The opening tick is aligned to the spacing and is the closest one at or below the exact
      price.
- [ ] The first buy charges the buy fee. Unspent ETH is refunded. The Factory holds no ETH or
      tokens afterwards.

## Token transfers

- [ ] Full supply minted once, to the Locker. No mint path afterwards.
- [ ] Transfers are not taxed.
- [ ] `eligibleSupply` equals the sum of non-excluded balances after every mint, transfer, buy and
      sell.
- [ ] Protocol addresses (Token, dead address, Rewards, Hook, PoolManager, Factory, Locker) hold no
      shares.

## Reward accounting

- [ ] Deposits are distributed pro-rata to `rewardWeightOf`.
- [ ] Transfers do not move rewards that were already earned.
- [ ] Sum of entitlements ≤ ETH received, and `outstandingEth() == address(rewards).balance`.
- [ ] Fees are held back (not lost) while `eligibleSupply < MIN_ELIGIBLE_SUPPLY`.

## 24h recovery

- [ ] The clock starts on first holding. Receiving more rewards or tokens does not restart it.
- [ ] Recovery is impossible at or before `lastActivityAt + 24h` and possible after it.
- [ ] Recovered ETH moves to `buybackReserve` in the same vault. Other holders' `pendingEth` is
      unchanged.
- [ ] Ineligible or duplicate accounts are skipped. The call reverts only when nothing was
      recovered.
- [ ] Only the buyback wallet can receive the reserve, via `claimAll`.

## Claim All

- [ ] Pays only `msg.sender`, across many tokens, in one call.
- [ ] Vaults with nothing owed return 0 without reverting. Unknown tokens revert with
      `UnknownToken`.
- [ ] `withdrawRewardsFor` and `recoverIdle` revert for any caller other than the Factory.
- [ ] A claim restarts the idle clock in each vault that paid out.

## Reentrancy

- [ ] A malicious holder contract that re-enters `claimAll`, `recoverIdle` or `launch` during
      payout is blocked.
- [ ] A holder contract that rejects ETH causes `PayoutReverted` without corrupting accounting.
- [ ] `unlockCallback` of the Factory and the Locker rejects callers other than the PoolManager
      and unexpected calls.

## Buy / sell fee routing

- [ ] Exact-input buy, exact-output buy and exact-input sell each charge the correct ETH fee.
- [ ] Exact-output sell reverts with `ExactOutputSellDisabled`.
- [ ] 100% of every fee reaches the pool's own Rewards in the same transaction. The Hook's balance
      stays zero.
- [ ] The LP fee stays 0. Fees can never change after registration.

## Multi-token isolation

- [ ] Fees from token A never reach token B's Rewards.
- [ ] Claims, recoveries and reserves in one vault never affect another.
- [ ] One pool per token (`AssetAlreadyListed`) and one lock per token (`TokenAlreadyLocked`).

## Permanent liquidity

- [ ] The locked position's liquidity never decreases after any sequence of swaps.
- [ ] The Locker exposes no function to remove, collect or transfer out.
- [ ] Locked position recorded with correct range, `placed + dust == amount`, and zero ETH side.

## Fuzz / invariant tests

- [ ] Fuzzed launch parameters (supply, market cap, fees): valid ones succeed, invalid ones revert
      cleanly.
- [ ] Fuzzed callers for privileged paths (buyback reserve, Factory-only functions).
- [ ] Invariants over random sequences of trades, transfers, claims, recoveries and time jumps:
      supply is constant, `eligibleSupply` is consistent, vaults stay solvent, launches stay
      isolated, and locked liquidity never moves.

## Mainnet-fork Uniswap v4 tests

- [ ] Deploy the full protocol against the canonical mainnet PoolManager on a fork.
- [ ] Mine a hook salt that is valid for the fork's CREATE2 deployer.
- [ ] Launch, swap through the canonical Universal Router or common aggregators, claim and recover
      on the fork.
- [ ] Confirm that gas for `launch`, swaps and `claimAll` is within expected limits.
