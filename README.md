# RatchetFloor

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/ratchet-floor?style=flat&logo=github)](https://github.com/nirholas/ratchet-floor/stargazers) [![License](https://img.shields.io/github/license/nirholas/ratchet-floor?style=flat)](https://github.com/nirholas/ratchet-floor/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/ratchet-floor?style=flat)](https://github.com/nirholas/ratchet-floor/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/ratchet-floor/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/ratchet-floor/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**A price floor that only ever moves up.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://ratchet-floor.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/RatchetFloorHook.sol`](src/hooks/RatchetFloorHook.sol)
- **Licence:** Apache-2.0

## How it works

A token's floor price is normally a promise: a treasury that says it will bid, a team that says it will buy back. Promises are only as good as the balance behind them and the people holding the keys. This hook makes the floor a property of the pool instead.

It records the highest tick the pool has ever reached and refuses to let the price settle more than `offsetTicks` below it: floor = max(floor, highWaterTick - offsetTicks) The floor is monotone by construction. It has no setter, no owner and no emergency path, so it cannot be lowered by anyone, including whoever deployed the pool. A rally raises it permanently; a decline never moves it.

How a swap meets the floor matters, so it is worth being precise. Uniswap v4 swaps already take a `sqrtPriceLimitX96`, and {sqrtPriceFloorX96} returns exactly the value to pass: a swap carrying it fills as much as the floor allows and stops there, which is the behaviour a seller wants. The `afterSwap` check is the backstop for callers that pass no limit, and for those the swap reverts rather than partially filling.

Routers should read the floor; the revert exists so that a router which does not cannot break the invariant. One tick is one basis point to within rounding, so `offsetTicks = 2000` is a floor twenty percent below the high.

## Prior art

Floor prices are usually a treasury commitment (protocol-owned liquidity, OHM-style backing) or a buyback hook that spends fees defending a level. Both depend on a balance and on whoever can move it. Enforcing a monotone floor as an invariant of the pool, with no treasury and no key, is a different construction: nothing is spent defending it and nothing can lower it.

## Where it does not help

This guarantees the pool will not print below the floor. It does not guarantee anyone can sell at the floor, because it holds no capital: once the price reaches the floor there is simply no more selling into the pool, and a holder who wants out has to wait for the price to recover or trade elsewhere. It converts a liquidity risk into a liquidity halt, honestly and predictably, but it does not make the risk disappear. A pool that needs a real bid at the floor needs a treasury behind it, and this is not that.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    RatchetFloorHook.Config({
        offsetTicks: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `offsetTicks` | `uint24` | ticks |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `BelowFloor(int24,int24)` | The swap would leave the pool below its floor. Pass `sqrtPriceFloorX96` as the swap's price limit. |
| `InvalidOffset()` | `offsetTicks` was zero, which would pin the floor to the current price and stop the pool trading down. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `afterSwap`

Mask: `0x1040`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # RatchetFloor
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # price-floor, launch, no-admin, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/ratchet-floor
cd ratchet-floor
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/ratchet-floor&type=Date)](https://www.star-history.com/#nirholas/ratchet-floor&Date)

<!-- three.ws:growth -->
## Support the project

If ratchet-floor saves you time, **[star it on GitHub](https://github.com/nirholas/ratchet-floor)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=ratchet-floor%3A%20A%20price%20floor%20that%20only%20ever%20moves%20up&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fratchet-floor) · [Share on Bluesky](https://bsky.app/intent/compose?text=ratchet-floor%3A%20A%20price%20floor%20that%20only%20ever%20moves%20up%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fratchet-floor) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fratchet-floor) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fratchet-floor&t=ratchet-floor%3A%20A%20price%20floor%20that%20only%20ever%20moves%20up) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fratchet-floor&title=ratchet-floor%3A%20A%20price%20floor%20that%20only%20ever%20moves%20up)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/ratchet-floor` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/ratchet-floor/issues) or [start a discussion](https://github.com/nirholas/ratchet-floor/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/ratchet-floor)](https://github.com/nirholas/ratchet-floor/graphs/contributors)

<!-- /three.ws:growth -->
