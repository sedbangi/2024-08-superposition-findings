[01] Users can perform 0-amount swaps
The `swap_2_internal` does not have a check if the `amount_0` and `amount_1` is > 0. Users can perform 0-amount swaps and update the `fee_growth_global_0` and `fee_growth_global_1` unnecessarily also the code follows the Uniswap V3 code where 0-amount swaps are not allowed so provide the same check here.

[02] Users can add 0 liquidity amount to a position with zero liquidity
`update_position` accepts `delta = 0` but the Uniswap V3 code has a [check](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Position.sol#L54) that makes the code revert if the user adds 0 liquidity to an empty position. This check is missing here and the function can be called only to update the `fee_growth_inside_0` and `fee_growth_inside_1`.

[03] There is no check if the `from` address is the current owner of the position
`transfer_position_E_E_C7_A3_C_D` transfers position from one address to another, however, it is not checked if the `from` address is the current owner of the position allowing foreign position to be transferred. The owner of the transferred position won't be removed and also `from` address position will be removed without actually being transferred.
