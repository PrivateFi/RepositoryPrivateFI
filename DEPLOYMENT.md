# Deployment

The high-level order for deploying and wiring PrivateFi on a new chain. This page contains no
private keys, secrets, RPC endpoints or private deployment data.

## Prerequisites

- The canonical Uniswap v4 `PoolManager` address on the target chain.
- A CREATE2 deployer on the target chain, for the Hook.
- The protocol buyback wallet address. It is **permanent**: stored as a private immutable in the
  Factory, with no setter.

## Order

| # | Step | Notes |
|---|---|---|
| 1 | **Token implementation** | `PrivateFiToken(bootstrapper = predicted Factory address)` |
| 2 | **Rewards implementation** | `PrivateFiRewards(bootstrapper = predicted Factory address)` |
| 3 | **LaunchFactory** | `PrivateFiLaunchFactory(tokenImpl, rewardsImpl, poolManager, buybackWallet)`. The constructor reverts unless both implementations name this Factory as `BOOTSTRAPPER`. The deployer becomes the owner and the one-time wirer. |
| 4 | **Locker** | `PrivateFiLocker(poolManager, factory)` |
| 5 | **Hook via CREATE2** | `PrivateFiHook(poolManager, initialRegistrar = deployer)`, deployed at a mined salt (see below) |
| 6 | **Hook registrar handoff** | `hook.handOffRegistrar(factory)`. One-time and irreversible. |
| 7 | **Factory wiring** | `factory.wire(hook, locker)`. One-time and irreversible. Checks the handoff and that all three components use the same PoolManager. |
| 8 | **Launch first token** | The genesis launch. Until it happens, `launch()` accepts only the Factory owner. After it, launches are permissionless. |

The implementations must name the Factory before the Factory exists. Their constructor argument is
therefore the Factory's predicted `CREATE` address (from the deployer's nonce). If the nonce moves
between the prediction and step 3, step 3 reverts.

Steps 4 and 5 do not depend on each other. The Locker needs only the PoolManager and the Factory,
and the Hook needs only the PoolManager and its temporary registrar. Both must exist before step 7.

## Hook address and permissions

Uniswap v4 reads a hook's permissions from the **low bits of its address**. PrivateFiHook enables:

| Permission | Enabled |
|---|---|
| `beforeInitialize` | ✓ |
| `beforeSwap` | ✓ |
| `afterSwap` | ✓ |
| `beforeSwapReturnDelta` | ✓ |
| `afterSwapReturnDelta` | ✓ |
| all others | — |

The Hook must therefore be deployed with CREATE2 at an address whose flag bits match exactly this
set. Mine the salt for the exact init code (creation code plus constructor arguments
`(poolManager, deployer)`) and the CREATE2 deployer being used. The Hook constructor validates its
own address and reverts on a mismatch.

## Handoff window

Between steps 5 and 6 the deployer is the Hook's registrar. Send nothing else from the deployer
key during that window. After deploying, check that `hook.poolCount() == 0`.

## Post-deployment checks

- `tokenImpl.BOOTSTRAPPER() == factory` and `rewardsImpl.BOOTSTRAPPER() == factory`
- `hook.factory() == factory` and `hook.registrar() == factory`
- `factory.hook() == hook`, `factory.locker() == locker`, `factory.isWired() == true`
- `locker.factory() == factory`
- `hook.v4PoolManager()`, `factory.poolManager()`, `locker.poolManager()` all equal the canonical PoolManager
- `factory.launchCount() == 0` and `factory.launchPaused() == false` before genesis
- `factory.owner()` is the intended owner. Consider moving ownership to a multisig with the
  two-step `transferOwnership` / `acceptOwnership`.

## Ownership notes

- The owner's only powers are the genesis launch and pausing new launches.
- Renouncing ownership before the genesis launch disables launches permanently.
- Renouncing afterwards permanently freezes the current pause state.
