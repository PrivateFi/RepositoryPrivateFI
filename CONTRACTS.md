# Contracts

This page covers the five production contracts. All function and event names match the ABIs in
[`abi/`](./abi). Implementation source is not published.

| Contract | Instances | ABI |
|---|---|---|
| [PrivateFiLaunchFactory](#privatefilaunchfactory) | 1 (shared) | [`abi/PrivateFiLaunchFactory.json`](./abi/PrivateFiLaunchFactory.json) |
| [PrivateFiHook](#privatefihook) | 1 (shared) | [`abi/PrivateFiHook.json`](./abi/PrivateFiHook.json) |
| [PrivateFiLocker](#privatefilocker) | 1 (shared) | [`abi/PrivateFiLocker.json`](./abi/PrivateFiLocker.json) |
| [PrivateFiToken](#privatefitoken) | 1 per launch (clone) | [`abi/PrivateFiToken.json`](./abi/PrivateFiToken.json) |
| [PrivateFiRewards](#privatefirewards) | 1 per launch (clone) | [`abi/PrivateFiRewards.json`](./abi/PrivateFiRewards.json) |

---

## PrivateFiLaunchFactory

### Purpose

The permissionless entry point of the protocol. It creates every launch in one atomic call, keeps
the registry (launch ID ↔ token ↔ rewards ↔ pool), and is the only way to reach Claim All and
idle recovery.

### Important public actions

| Function | Who | Description |
|---|---|---|
| `launch(LaunchParams)` *payable* | Anyone (the owner for the genesis launch) | Creates a Token, Rewards and pool, locks liquidity and makes an optional first buy with `msg.value`. Returns `(launchId, token, rewards, poolId)`. |
| `claimAll(address[] tokens)` | Anyone | Pays the caller's own ETH rewards from each listed token's vault. |
| `recoverIdle(address token, address[] accounts)` | Anyone (keeper) | Moves rewards idle for more than 24 hours into that token's buyback reserve. |
| `openPriceFor(supply, marketCapWei)` *pure* | Anyone | Previews the opening tick and `sqrtPriceX96` for a launch. |
| `wire(hook, locker)` | Deployer, once | Connects the Hook and Locker. Irreversible. |
| `setLaunchPaused(bool)` | Owner | Stops or resumes **new** launches only. |
| `transferOwnership` / `acceptOwnership` / `renounceOwnership` | Owner / pending owner | Two-step ownership (Ownable2Step). |

Views: `launchCount`, `launches`, `getLaunch`, `getLaunchByToken`, `getLaunches` (paginated,
`MAX_PAGE = 100`), `launchIdOf`, `rewardsOf`, `isLaunched`, `poolKeyOf`, `launchCountOf`,
`launchIdsOf`, `pendingRewards(tokens, account)`, `isWired`, `launchPaused`, `hook`, `locker`,
`poolManager`, `TOKEN_IMPLEMENTATION`, `REWARDS_IMPLEMENTATION`, plus the bound constants
(`MIN_LAUNCH_SUPPLY`, `MAX_LAUNCH_SUPPLY`, `MAX_FEE_BPS`, `MIN_OPEN_TICK`, `MAX_OPEN_TICK`,
`MAX_NAME_LENGTH`, `MAX_SYMBOL_LENGTH`, `MAX_METADATA_URI_LENGTH`, `LP_FEE`, `POOL_TICK_SPACING`).

Events: `TokenLaunched`, `FirstBuy`, `ClaimAll`, `IdleRecovered`, `FactoryWired`, `LaunchPausedSet`,
and the ownership events.

### Trusted callers

- **Uniswap v4 PoolManager:** the only caller allowed into `unlockCallback`, which is used for the
  first buy.
- **Deployer (wirer):** can call `wire` exactly once.
- **Owner:** can call `setLaunchPaused` and make the genesis launch.

### Security restrictions

- The owner **cannot** touch existing tokens, pools, fees, rewards, reserves or liquidity. The
  pause only blocks new launches. Trading, fees, `claimAll` and `recoverIdle` keep working while
  paused.
- There are no setters for the Hook, Locker or implementations after `wire`, and no withdrawal
  function. The Factory never holds ETH or tokens between calls.
- The constructor checks that both implementations name this Factory as their `BOOTSTRAPPER`.
- `wire` checks that the Hook's registrar role has already been handed to this Factory and that
  the Hook, Locker and Factory all use the same PoolManager.
- All launch parameters are bounded: supply, fees ≤ 10%, opening tick range, and string lengths.
- `launch`, `claimAll` and `recoverIdle` are `nonReentrant` (transient-storage guard).
- The buyback wallet is a private immutable, fixed at deployment. It has no getter, and no event
  names it (see [SECURITY.md](./SECURITY.md#privacy-clarification)).

---

## PrivateFiHook

### Purpose

The single Uniswap v4 hook for every PrivateFi pool. It stores each pool's immutable fee config,
charges the buy or sell fee in ETH on every swap, and forwards 100% of the fee to that pool's
Rewards in the same call.

### Important public actions

| Function | Who | Description |
|---|---|---|
| `registerPool(key, vault, buyFeeBps, sellFeeBps, sqrtPriceX96)` | Registrar (the Factory) | Validates the pool, stores the immutable config and initializes the pool on v4. |
| `handOffRegistrar(factory)` | Initial registrar, once | Hands the registrar role to the Factory permanently. |
| `beforeInitialize` / `beforeSwap` / `afterSwap` | PoolManager | Hook callbacks. |

Views: `poolConfig(poolId)`, `feesOf(poolId)`, `poolIdOf(token)`, `poolCount`, `registrar`,
`factory`, `v4PoolManager`, `getHookPermissions`, `MAX_FEE_BPS` (1000 = 10%), `BPS_DENOMINATOR`.

Events: `PoolListed`, `SwapFeeRouted`, `RegistrarHandedOff`.

> The ABI also lists the other `IHooks` callbacks (`afterInitialize`, `beforeAddLiquidity`, …),
> which come from the base contract. They are not enabled in the hook's permission bits, so the
> PoolManager never calls them.

### Trusted callers

- **Uniswap v4 PoolManager:** the only caller of the callbacks and the only sender of ETH to the
  Hook.
- **Registrar:** the deployer temporarily, then the LaunchFactory permanently after
  `handOffRegistrar`.

### Security restrictions

- A pool can only be created through `registerPool`. `beforeInitialize` rejects every other
  initializer.
- Pool configs are write-once. No function changes fees or the Rewards address.
- One pool per token (`AssetAlreadyListed`).
- Pool key requirements: `currency0 = ETH`, `currency1 = token`, `hooks = this`, dynamic-fee flag
  required (the LP fee stays 0 forever).
- Pairing check: `rewards.asset() == token`, `rewards.feeHook() == hook`, `token.vault() == rewards`.
  The token must exclude both the Hook and the PoolManager from reward shares.
- Fees cannot exceed `MAX_FEE_BPS`.
- Exact-output sells revert (`ExactOutputSellDisabled`).
- The Hook keeps no fee balance. Every fee it takes is forwarded within the same callback. There is
  no platform share.
- It must be deployed at a CREATE2 address whose low bits encode its permissions. The constructor
  verifies this.

---

## PrivateFiLocker

### Purpose

The single, permanent custodian of every launch's liquidity. The lock works because the contract
has no code path to remove anything.

### Important public actions

| Function | Who | Description |
|---|---|---|
| `lock(key, openTick, amount)` | Factory only | Adds the token supply held by the Locker as one single-sided position. |
| `unlockCallback(data)` | PoolManager only | Internal leg of `lock`. |

Views: `positionOf(poolId)`, `isLocked(poolId)`, `lockedPoolOf(token)`, `lockCount`,
`rangeFor(spacing, openTick, tokenIsCurrency0)`, `factory`, `poolManager`.

Events: `LiquidityLocked`.

### Trusted callers

- **PrivateFiLaunchFactory** (immutable): the only caller of `lock`.
- **Uniswap v4 PoolManager**: the only caller of `unlockCallback`, and only while a `lock` call is
  in progress (transient flag).

### Security restrictions

- It has no function that removes liquidity, collects fees, moves a position or transfers tokens
  out. No owner, no rescue, no upgrade.
- One lock per pool and one per token.
- The pool price must equal the opening tick exactly (`PriceMismatch`), and the ETH side of the
  position must be zero (`NotOneSided`).
- Rounding dust (a few wei of the supply) stays in the Locker forever. The Locker holds no reward
  shares.

---

## PrivateFiToken

### Purpose

A fixed-supply ERC-20, cloned once per launch. Its balances are the share ledger of that token's
Rewards vault.

### Important public actions

| Function | Who | Description |
|---|---|---|
| `transfer`, `transferFrom`, `approve` | Holders | Standard ERC-20. No transfer tax. |
| `bootstrap(TokenInitParams)` | `BOOTSTRAPPER` (the Factory), once | Sets metadata and exclusions, then mints the full supply. |

Views: standard ERC-20 (`name`, `symbol`, `decimals` = 18, `totalSupply`, `balanceOf`, `allowance`),
plus `rewardWeightOf`, `eligibleSupply`, `isRewardExempt`, `vault`, `feeHook`, `creator`,
`v4PoolManager`, `factory`, `locker`, `bootstrapped`, `BOOTSTRAPPER`, `BURN_SINK`, `MIN_SUPPLY`,
`MAX_SUPPLY`, `MAX_NAME_LENGTH`, `MAX_SYMBOL_LENGTH`.

Events: `Transfer`, `Approval`, `Bootstrapped`, `RewardExempted`.

### Trusted callers

- **`BOOTSTRAPPER`:** an immutable of the implementation, shared by all clones. It is set to the
  LaunchFactory, so only the Factory can initialize a clone.
- The Token calls only its own Rewards (`onBalanceShift`, `onWeightRemoved`).

### Security restrictions

- Minting happens only inside `bootstrap`, which can run once. No mint function exists afterwards.
- No owner, admin, blacklist, pause, or setter after initialization.
- `creator` is informational only and gives no privileges.
- Protocol addresses are excluded from reward shares before the mint, so they never hold a share.
- Supply is bounded by `MIN_SUPPLY` (1,000 tokens) and `MAX_SUPPLY` (`type(uint96).max`). The
  Factory applies tighter launch bounds.

---

## PrivateFiRewards

### Purpose

The isolated ETH reward vault of a single token. It receives all fees from that token's pool,
shares them out pro-rata to eligible holders, and holds that token's buyback reserve.

### Important public actions

| Function | Who | Description |
|---|---|---|
| `depositFees()` *payable* | Hook only | Distributes `msg.value` across all live shares. Sends nothing out. |
| `onBalanceShift(from, to, value)` | Its Token only | Keeps earned rewards with their owner across transfers. |
| `onWeightRemoved(account, weight)` | Its Token only | The account stops holding shares but keeps what it earned. |
| `withdrawRewardsFor(account, isBuybackWallet)` | Factory only | Pays `account` its own ETH. Pays the reserve too when the Factory flags its buyback wallet. |
| `recoverIdle(address[] accounts)` | Factory only | Moves balances idle for more than 24h into `buybackReserve`. |
| `bootstrap(asset, feeHook)` | `BOOTSTRAPPER`, once | Binds the vault to its token and the Hook. |

Views: `pendingEth`, `accruedEth`, `recoverableAt`, `isRecoverable`, `lastActivityAt`, `withdrawn`,
`weightCorrection`, `accEthPerWeight`, `buybackReserve`, `outstandingEth`, `carryEth`,
`carryScaled`, `lifetimeDeposits`, `lifetimeWithdrawn`, `lifetimeRecovered`,
`lifetimeReserveReleased`, `asset`, `feeHook`, `factory`, `IDLE_WINDOW`, `MIN_ELIGIBLE_SUPPLY`.

Events: `FeesDeposited`, `RewardsWithdrawn`, `IdleRewardsRecovered`, `BuybackReserveGrew`,
`BuybackReserveReleased`, `Bootstrapped`.

### Trusted callers

- **PrivateFiHook** (`feeHook`): the only depositor.
- **Its PrivateFiToken** (`asset`): the only source of ledger updates.
- **PrivateFiLaunchFactory:** resolved at call time via `Hook.factory()`, which is set once at the
  registrar handoff. It is the only caller of `withdrawRewardsFor` and `recoverIdle`. Before the
  handoff, nobody can call either.

### Security restrictions

- ETH leaves only through `withdrawRewardsFor`, and only to the account that is owed. There is no
  recipient parameter.
- No owner, sweep, rescue or setter.
- `withdrawRewardsFor` is `nonReentrant`, and all accounting is written before ETH is sent.
- `recoverIdle` moves no ETH out and does not change the accumulator, so other holders are
  unaffected.
- `outstandingEth()` always equals the contract's ETH balance. Rounding remainders are carried
  forward, so the vault never owes more than it received.
- It holds no state about any other launch.
