# 🥪 Sandwich Resistant Hook

[![CI][ci-badge]][ci-url]

Uniswap v4 hook that is resistant to atomic sandwich attacks.

## Installation

To install with [**Foundry**](https://github.com/foundry-rs/foundry):

```sh
forge install cairoeth/sandwich-resistant-hook
```

## Design

This hook implements the sandwich-resistant AMM design introduced in [A Sandwich Resistant AMM](https://www.umbraresearch.xyz/writings/sandwich-resistant-amm). Specifically, this hook guarantees that no swaps get filled at a price better than the price at the beginning of the slot window (i.e. one block):

> Within a slot window, swaps impact the pool asymmetrically for buys and sells. When a buy order is executed, the offer on the pool increases in accordance with the xy=k curve. However, the bid price remains constant, instead increasing the amount of liquidity on the bid. Subsequent sells eat into this liquidity, while decreasing the offer price according to xy=k.

In addition to the points noted in the [`Considerations`](https://www.umbraresearch.xyz/writings/sandwich-resistant-amm#considerations) section of the article, it's worth noting that swaps in the other direction do not get the positive price difference compared to the initial price before the first block swap.

## Implementation

https://github.com/user-attachments/assets/9be3d43c-973d-4f95-8e18-486bd897207f

The design is implemented on top of [`base-v4-hook`](https://github.com/cairoeth/base-v4-hook) to act as a no-op hook, which allows to skip the swap logic from the Uniswap v4 `PoolManager` contract, while implementing v4-like liquidity logic. In order to be able to reset the pool state after each slot window, the hook tracks the base pool state, and a temporary pool state. Before executing the first swap in a slot window, the hook checkpoints the initial `slot0` from the base pool, which contains the current price, tick, protocol fee, and lp fee. Then, after executing the first swap on the base pool state, the hook initializes the temporary pool state as a copy of the base pool state. Subsequent swaps in the same slot window execute the swap on both the base and temporary pool states, but the token deltas are applied only based on the output from the temporary pool state. Depending on the swap direction, the hook applies the checkpointed `slot0` to the temporary pool state to ensure that the bid price remains constant and that liquidity on the bid tick increases as buy orders are executed.

In the first swap of the next slot window, the hook checkpoints the new `slot0` from the base pool state, executes the swap on the base pool state, and re-initializes the temporary pool state.

The main gas trade-offs arise from initializing the temporary pool state, and executing the swap logic twice for each swap after the first one in a slot window. An optimization is performned when initializing the temporary pool state by only copying storage values that get modified during the swap logic. Further gas optimizations can be made by tracking specific swap value differences instead of executing the swap logic completely twice. Another area to explore is how liquidity modifications can affect the implementation details and logic.

## Acknowledgements

This repository is inspired by or directly modified from many sources, primarily:

- [A Sandwich Resistant AMM](https://www.umbraresearch.xyz/writings/sandwich-resistant-amm)
- [base-v4-hook](https://github.com/cairoeth/base-v4-hook)

[ci-badge]: https://github.com/cairoeth/sandwich-resistant-hook/actions/workflows/test.yml/badge.svg
[ci-url]: https://github.com/cairoeth/sandwich-resistant-hook/actions/workflows/test.yml
