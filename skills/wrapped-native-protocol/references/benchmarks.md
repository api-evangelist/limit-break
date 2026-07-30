# Wrapped Native Gas Benchmarks

All benchmarks compare WNATIVE against WETH9 using identical test conditions.

## Per-Function Gas Savings

| Function | Gas Saved (WNATIVE vs WETH9) |
|---|---|
| balanceOf | 88 |
| native transfer (deposit) | 48 |
| deposit | 108 |
| withdraw | 395 |
| approve | 213 |
| transfer | 627 |
| transferFrom (self transfer) | 481 |
| transferFrom (operator allowance) | 1,080 |
| transferFrom (operator unlimited) | 508 |

## Aggregate L1 Savings (ETH Mainnet, Sep 2022 - Oct 2024)

Average ETH price during period: $2,189.25

| Operation | Savings/Op | Mainnet Operations | Gas Units Saved | ETH Saved |
|---|---|---|---|---|
| balanceOf | 88 | 327,453,459 | 29B | 626 ETH |
| transfer | 627 | 179,154,025 | 112B | 2,419 ETH |
| withdraw | 395 | 61,370,490 | 24B | 518 ETH |
| transferFrom | 794 | 24,489,452 | 19.5B | 421 ETH |
| approve | 213 | 6,936,611 | 1.5B | 32 ETH |
| deposit | 108 | 125,633 | 13.5M | ~0 ETH |
| **Total** | | | **186B** | **4,016 ETH (~$8.8M)** |

Average post-merge gas price: 21.6 gwei

The official WNATIVE address contains 8 zero-bytes, providing an additional 96 gas in calldata savings per call. Accounting for this, an estimated additional $1.2M would have been saved.

Benchmarks cover only interfaces shared between WETH9 and WNATIVE. Features unique to WNATIVE (depositTo, withdrawSplit, permits) are not included since they have no WETH9 equivalent.
