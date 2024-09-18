| Count | Title |
| --- | --- |
| QA-01 | Operator cannot approve via OwnershipNFTs |
| QA-02 | `get_liquidity_for_amounts` is lowering from `u256` to `int128` |
| QA-03 | Sqrt ratios won’t overflow on time due to u256 being used instead of u128 |

| Total Issues | 3 |
| --- | --- |

## [QA-01] Operator cannot approve via OwnershipNFTs

**Issue Description:**

An operator cannot approve an NFT from an `OwnershipNFT` because the approver passes `msg.sender` as `from` to `_requireAuthorised()`, which strictly checks that `from` is the `owner` of the item.

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160-L163

```solidity
function approve(address _approved, uint256 _tokenId) external payable {
    _requireAuthorised(msg.sender, _tokenId);
    getApproved[_tokenId] = _approved;
}
```

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L98-L107

```solidity
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
    // revert if the sender is not authorised or the owner
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "not allowed");
    require(ownerOf(_tokenId) == _from, "_from is not the owner!");
}
```

But this is not the case in `OpenZeppelin's` implementation and how it is defined in the EIP. When an operator is `approved` to act on a user's NFT, he should be able to `approve` other addresses as well, because the whole point of operators is useless if this check doesn't allow it to do anything.

**Recommendation:**

Allow the operator to approve other addresses.

## [QA-02] `get_liquidity_for_amounts` is lowering from `u256` to `int128`

**Issue Description:**

The result of `get_liquidity_for_amounts()` is cast to `int128` from `u256` inside `adjust_position()`, but this will reduce the possible output of `get_liquidity_for_amounts()` to **`6.81 x 10^38`** times lower than `u256` max. In fact, the output of this needs to be converted to a signed integer because it is the delta liquidity that will be changed and it can be positive or negative. But the result of `get_liquidity_for_amounts` is `u256` and when it is cast to `int128` the value that can be stored there will become **`6.81 x 10^38`** times less because `u256` can hold `2^256 -1` while `int128` can `2^127 -1` .

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/pool.rs#L257-L281

```rust
pub fn adjust_position(
    &mut self,
    id: U256,
    amount_0: U256,
    amount_1: U256,
    giving: bool,
) -> Result<(I256, I256), Revert> {
    // calculate the delta using the amounts that we have here, guaranteeing
    // that we don't dip below the amount that's supplied as the minimum.

    let position = self.positions.positions.get(id);

    let sqrt_ratio_x_96 = tick_math::get_sqrt_ratio_at_tick(self.get_cur_tick().as_i32())?;
    let sqrt_ratio_a_x_96 = tick_math::get_sqrt_ratio_at_tick(position.lower.get().as_i32())?;
    let sqrt_ratio_b_x_96 = tick_math::get_sqrt_ratio_at_tick(position.upper.get().as_i32())?;

    let mut delta = sqrt_price_math::get_liquidity_for_amounts(
        sqrt_ratio_x_96,   // cur_tick
        sqrt_ratio_a_x_96, // lower_tick
        sqrt_ratio_b_x_96, // upper_tick
        amount_0,          // amount_0
        amount_1,          // amount_1
    )?
    .to_i128() // <_-------------------------
    .map_or_else(|| Err(Error::LiquidityAmountTooWide), Ok)?;
```

**Recommendation:**

To reduce the value boundaries from `2^256 - 1` to `2^255 -1`, just use `int256` instead of `int128`.

## [QA-03] Sqrt ratios won’t overflow on time due to u256 being used instead of u128

**Issue Description:**

Unlike `UniswapV3`, where `sqrtRatioCurrentX96` and `sqrtRatioTargetX96` in `SwapMath::computeSwapStep` are of type uint160 and all the calculations are in unchecked block, in Superposition these same variables are of type u256 and use checked math:

[swap_math.rs](https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/maths/swap_math.rs#L23-L155)

```rust
pub fn compute_swap_step(
    sqrt_ratio_current_x_96: U256, //LEAD these won't overflow on time, they're uint160 in UniV3 - https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/SwapMath.sol#L22-L23
    sqrt_ratio_target_x_96: U256,
    liquidity: u128,
    amount_remaining: I256,
    fee_pips: u32,
) -> Result<(U256, U256, U256, U256), Error> {
    let zero_for_one = sqrt_ratio_current_x_96 >= sqrt_ratio_target_x_96;
    let exact_in = amount_remaining >= I256::zero();

    let sqrt_ratio_next_x_96: U256;
    let mut amount_in = U256::zero();
    let mut amount_out = U256::zero();

    if exact_in {
        let amount_remaining_less_fee = mul_div(
            amount_remaining.into_raw(),
            U256::from(1e6 as u32 - fee_pips), //1e6 - fee_pips
            U256::from(1e6 as u32),            //1e6
        )?;

        amount_in = if zero_for_one {
            _get_amount_0_delta(
                sqrt_ratio_target_x_96,
                sqrt_ratio_current_x_96,
                liquidity,
                true,
            )?
        } else {
            _get_amount_1_delta(
                sqrt_ratio_current_x_96,
                sqrt_ratio_target_x_96,
                liquidity,
                true,
            )?
        };

        if amount_remaining_less_fee >= amount_in {
            sqrt_ratio_next_x_96 = sqrt_ratio_target_x_96;
        } else {
            sqrt_ratio_next_x_96 = get_next_sqrt_price_from_input(
                sqrt_ratio_current_x_96,
                liquidity,
                amount_remaining_less_fee,
                zero_for_one,
            )?;
        }
    } else {
        amount_out = if zero_for_one {
            _get_amount_1_delta(
                sqrt_ratio_target_x_96,
                sqrt_ratio_current_x_96,
                liquidity,
                false,
            )?
        } else {
            _get_amount_0_delta(
                sqrt_ratio_current_x_96,
                sqrt_ratio_target_x_96,
                liquidity,
                false,
            )?
        };

        sqrt_ratio_next_x_96 = if (-amount_remaining).into_raw() >= amount_out {
            sqrt_ratio_target_x_96
        } else {
            get_next_sqrt_price_from_output(
                sqrt_ratio_current_x_96,
                liquidity,
                (-amount_remaining).into_raw(),
                zero_for_one,
            )?
        };
    }

    let max = sqrt_ratio_target_x_96 == sqrt_ratio_next_x_96;

    if zero_for_one {
        if !max || !exact_in {
            amount_in = _get_amount_0_delta(
                sqrt_ratio_next_x_96,
                sqrt_ratio_current_x_96,
                liquidity,
                true,
            )?
        }

        if !max || exact_in {
            amount_out = _get_amount_1_delta(
                sqrt_ratio_next_x_96,
                sqrt_ratio_current_x_96,
                liquidity,
                false,
            )?
        }
    } else {
        if !max || !exact_in {
            amount_in = _get_amount_1_delta(
                sqrt_ratio_current_x_96,
                sqrt_ratio_next_x_96,
                liquidity,
                true,
            )?
        }

        if !max || exact_in {
            amount_out = _get_amount_0_delta(
                sqrt_ratio_current_x_96,
                sqrt_ratio_next_x_96,
                liquidity,
                false,
            )?
        }
    }

    if !exact_in && amount_out > (-amount_remaining).into_raw() {
        amount_out = (-amount_remaining).into_raw();
    }

    if exact_in && sqrt_ratio_next_x_96 != sqrt_ratio_target_x_96 {
        let fee_amount = amount_remaining.into_raw() - amount_in;
        Ok((sqrt_ratio_next_x_96, amount_in, amount_out, fee_amount))
    } else {
        let fee_amount = mul_div_rounding_up(
            amount_in,
            U256::from(fee_pips),
            U256::from(1e6 as u32 - fee_pips),
        )?;

        Ok((sqrt_ratio_next_x_96, amount_in, amount_out, fee_amount))
    }
}
```

All the math functions here will prevent overflow, whereas the same functionality in `UniswapV3` expects them to happen. 

The more important issue is that `sqrt_ratio_target_x_96` and `sqrt_ratio_current_x_96` are of type uint256, which will completely prevent them from underflowing and will execute the swaps with wrong prices when they become higher than `uint160.max`, this is critical for the tick based AMMs like `UniV3` and `Superposition`.

**Recommendation:**

1. Use unchecked math across the entire function
2. Use sqrt ratios with types `u160` instead of `U256`, making them vulnerable to overflows in order to provide the users with proper prices.