# Security

This page describes the security properties the PrivateFi contracts enforce, the access-control
model, and the risks that remain. It is not an audit report, and no audit report is included in
this repository.

## Reporting a vulnerability

Please report vulnerabilities privately through the official channels listed on
[privatefi.app](https://privatefi.app) or via DM to [@privatefihooks](https://x.com/privatefihooks).
Do not open a public issue for a suspected vulnerability.

---

## Fixed supply

- Each token's full supply is minted exactly once, inside `PrivateFiToken.bootstrap`, which can
  only run once and only from the LaunchFactory.
- The whole supply is minted straight to the Locker and placed in the pool at launch. The creator
  receives no tokens except through an optional first buy, which pays the normal market price and
  buy fee.

## No post-launch mint

- The token has no mint function, owner, admin role, pause, blacklist or upgrade path.
- Token clones are EIP-1167 minimal proxies that point to a fixed implementation. There is no
  proxy admin.

## Permanent liquidity

- `PrivateFiLocker` has no function that removes liquidity, collects fees, moves a position or
  transfers tokens out. It has no owner and cannot be upgraded.
- Only the Factory can lock, once per pool and once per token, during the launch transaction.
- LP positions earn nothing: the pool's LP fee is fixed at 0, so there are no LP fees to collect.

## Isolated reward vaults

- Every launch has its own `PrivateFiRewards` clone, bound once to one token and the Hook.
- The Hook sends each pool's fees only to the vault recorded in that pool's write-once config.
- A vault accepts deposits only from the Hook and ledger updates only from its own token.
- The buyback reserve is kept separately in each token's own vault.

## Immutable pool fee configuration

- Buy and sell fees are chosen by the creator at launch, capped at `MAX_FEE_BPS` = 1000 (10%).
- `PrivateFiHook.registerPool` writes the config once. No function changes fees or the Rewards
  address afterwards, and each token can have only one pool.
- Pools use the dynamic-fee flag with an LP fee of 0. Only the hook could change a dynamic LP fee,
  and PrivateFiHook contains no code that does. The hook fee is the only swap fee.
- Nobody can create pools on the hook except through `registerPool`. `beforeInitialize` rejects
  outside initializers.

## Access-control model

| Actor | Powers | Cannot |
|---|---|---|
| **Anyone** | `launch` (after genesis), `claimAll` for themselves, `recoverIdle` as keeper | Claim for someone else, redirect rewards, change any pool |
| **Factory owner** | `setLaunchPaused` (new launches only), genesis launch, two-step ownership transfer or renounce | Mint, change fees, touch liquidity, move rewards or reserves |
| **Deployer (wirer)** | Call `wire` once. As temporary Hook registrar, call `handOffRegistrar` once | Anything after wiring. There are no setters |
| **Token creator** | Pick name, symbol, supply, fees and opening market cap within bounds, at launch only | Anything after launch. `creator` gives no privileges |
| **Buyback wallet** | Receives each vault's buyback reserve when it calls `claimAll` | Take holders' unclaimed rewards before they are idle for 24h |
| **LaunchFactory (contract)** | Hook registrar. Only caller of `lock`, `withdrawRewardsFor`, `recoverIdle` | Holds no funds between calls |
| **PrivateFiHook (contract)** | Only depositor into Rewards | Keep fees. Every fee is forwarded in the same call |
| **PoolManager** | Only caller of hook callbacks and unlock callbacks | — |

Contract-to-contract trust is fixed at deployment or at one-time initialization:

- Implementations name the Factory as their immutable `BOOTSTRAPPER`.
- The Hook hands its registrar role to the Factory once, irreversibly.
- Rewards resolves the Factory through `Hook.factory()`, which is set only by that handoff.

## Reentrancy considerations

- **Factory:** `launch`, `claimAll` and `recoverIdle` are protected by a transient-storage
  reentrancy guard. `unlockCallback` accepts only the PoolManager.
- **Rewards:** `withdrawRewardsFor` is `nonReentrant` and follows checks-effects-interactions.
  `withdrawn`, the totals and the idle clock are updated before ETH is sent. `depositFees` only
  updates the accumulator and sends nothing, so a swap never calls into a holder.
- **Hook:** callbacks are restricted to the PoolManager. The only external call during a swap is
  `depositFees` on the pool's own vault, which is trusted and makes no outbound calls.
- **Locker:** `unlockCallback` is accepted only from the PoolManager and only while `lock` has set
  a transient flag.
- **Token:** every `_update` calls into its own Rewards after balances and `eligibleSupply` are
  written, so Rewards always sees the post-transfer state.

## Protocol risks

- **Smart-contract risk.** Bugs may exist despite testing. The contracts are not upgradeable, so a
  defect cannot be patched in place.
- **Uniswap v4 dependency.** Pools, prices and liquidity live in the canonical Uniswap v4
  PoolManager. A failure there affects every PrivateFi pool.
- **Untrusted launches.** Anyone can launch. Names, symbols and metadata URIs are not verified and
  may imitate other projects. Always check the token address and the Factory registry.
- **Fees up to 10%.** Each token has its own fees, up to 10% per side. Check `feesOf(poolId)`
  before trading.
- **Idle recovery.** Rewards unclaimed for more than 24 hours since an account's last claim or
  recovery (or since it first held the token) can be moved by anyone into that token's buyback
  reserve. Holders who want to keep their rewards must claim regularly.
- **Buyback reserve use.** Recovered ETH is released only to the protocol's buyback wallet. How
  that ETH is used after release depends on the operator and is not enforced by the contracts.
- **Payout to contracts.** If a claiming account is a contract that rejects ETH, its claim reverts
  (`PayoutReverted`), and the whole `claimAll` batch reverts with it.
- **Low eligible supply.** While fewer than 1 token's worth of shares is held outside excluded
  addresses, fees are held back and paid out in a later distribution.
- **Market risk.** Prices can move sharply. Permanent liquidity does not protect against losses.

## Privacy clarification

PrivateFi's "private" means a **private-by-default UX**: positions and rewards are shown to their
owner, and no feature publishes or ranks individual holders.

It does **not** mean anonymity. PrivateFi runs on public Ethereum:

- Every launch, swap, transfer, claim and recovery is a public transaction.
- Balances, reward accounting (`pendingEth`, `lastActivityAt`, …) and events can be read by anyone.
- The buyback wallet has no getter and no event names it, but it can still be found from
  deployment calldata, bytecode immutables and transaction senders.

PrivateFi does not use mixers, shielded pools or zero-knowledge privacy, and makes no anonymity
claims.
