### Low-01 Inconsistent overflow handling of sqrt price may cause unexpected swap behaviors
**Instances(1)**
In UniswapV3 sqrt price (a Q64.96) number is represented as uint160, when performing price calculation, uint256 is used and uint256 -> uin160 safe down case is always checked when needed, and it ensures revert when down casting is unsafe.

For example, in UniswapV3's token1 -> token0 flow (sqrt price increases), safe downcast to uint160 is always checked. see [getNextSqrtPriceFromAmount0RoundingUp()](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/SqrtPriceMath.sol#L54) and [getNextSqrtPriceFromAmount1RoundingDown()](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/SqrtPriceMath.sol#L84).

However, in the rust version, one instance of uint160 safe downcast check is missed. In `get_next_sqrt_price_from_amount_0_rounding_up()`, there is no safe uint160 downcast check. 
```rust
pub fn get_next_sqrt_price_from_amount_0_rounding_up(
    sqrt_price_x_96: U256,
    liquidity: u128,
    amount: U256,
    add: bool,
) -> Result<U256, Error> {
...
    } else {
        let product = amount.wrapping_mul(sqrt_price_x_96);
                if product.wrapping_div(amount) == sqrt_price_x_96 && numerator_1 > product {
            let denominator = numerator_1.wrapping_sub(product);
                        //@audit missing check the value <= MAX_U160
|>                      mul_div_rounding_up(numerator_1, sqrt_price_x_96, denominator)
        } else {
            Err(Error::ProductDivAmount)
        }
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/maths/sqrt_price_math.rs#L97)

For comparison, in the other instance `get_next_sqrt_price_from_amount_1_rounding_down`. U160 safe down cast check is in place.
```rust
pub fn get_next_sqrt_price_from_amount_1_rounding_down(
    sqrt_price_x_96: U256,
    liquidity: u128,
    amount: U256,
    add: bool,
) -> Result<U256, Error> {
...
    if add {
        let quotient = if amount <= MAX_U160 {
            (amount << FIXED_POINT_96_RESOLUTION) / liquidity
        } else {
            mul_div(amount, Q96, liquidity)?
        };

        let next_sqrt_price = sqrt_price_x_96 + quotient;
        //@audit-info note: safe down cast is checked here.
|>      if next_sqrt_price > MAX_U160 {
            Err(Error::SafeCastToU160Overflow)
        } else {
            Ok(next_sqrt_price)
        }
...
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/maths/sqrt_price_math.rs#L122)

Recommendations:
In `get_next_sqrt_price_from_amount_0_rounding_up()` ‘s  else branch, add a check to revert when `mul_div_rounding_up(numerator_1, sqrt_price_x_96, denominator)` > MAX_U160

### Low-02 fUSDC rewards will likely be locked in AMM’s proxy.
**Instances(1)**
According to [superposition doc](https://docs.superposition.so/native-dapps/longtail-amm), Longtail AMM engages with superposition chain's utility mining and supper assets rewards to enable yield for every swap. 
>It is also Arbitrum's cheapest and most rewarding AMM. Using Super Assets, every swap is eligible for yield.

This also means that seawaterAMM's proxy contract is eligible for rewards. This is because supper assets(e.g.fUSDC, fluid assets) reward both the sender address and receiver address of a token transfer. So SeawaterAMM.sol proxy, as sender/receiver of fUSDC transfers is eligible for fUSDC rewards.

However, the problem is SeawaterAMM.sol and its implementation facets are not able to collect the rewards minted.

Based on [fluid assets' doc](https://docs.fluidity.money/docs/fundamentals/faq#how-are-rewards-distributed), rewards are distributed on-chain by permissioned fluid asset minting. In seawaterAMM's case, an off-chain worker will call [fUSDC::batchReward() -> mint(winner, amount)](https://github.com/fluidity-money/fluidity-app/blob/07d62d3164ceb427fa3bda73b99c3ada683fbe9e/contracts/ethereum/contracts/Token.sol#L710-L732). This [mint fUSDC](https://github.com/fluidity-money/fluidity-app/blob/07d62d3164ceb427fa3bda73b99c3ada683fbe9e/contracts/ethereum/contracts/Token.sol#L710-L732) to seawaterAMM.sol's balance.

The directly minted fUSDC or other reward tokens to SeawaterAMM.sol proxy will Not be accounted for automatically in swapping because UniswapV3's logic doesn't use `token1.balanceOf`. We see currently there is no function in the deployed facets(swap, positions, admin) that allows the admin to transfer out the reward tokens. Rewards will be locked in the proxy.

Recommendations:
In the admin facet, add an admin-only function to transfer fUSDC (or other reward tokens) out. The rewarded amount can be calculated off-chain.

### Low-03 Risk of `mint_position_B_C5_B086_D` spamming with 0 liquidity for profits
**Instances(1)**
`mint_position_B_C5_B086_D` will simply mint a position tokenId for anyone who calls it. It doesn't require any liquidity deposit. THis means a caller can easily spam call `mint_position_B_C5_B086_D` with marginal gas cost. 
```rust
//pkg/seawater/src/lib.rs
    pub fn mint_position_B_C5_B086_D(
        &mut self,
        pool: Address,
        lower: i32,
        upper: i32,
    ) -> Result<U256, Revert> {
        let id = self.next_position_id.get();
        self.pools.setter(pool).create_position(id, lower, upper)?;
        self.next_position_id.set(id + U256::one());
        let owner = msg::sender();
|>      self.grant_position(owner, id);
        #[cfg(feature = "log-events")]
        evm::log(events::MintPosition {
            id,
            owner,
            pool,
            lower,
            upper,
        });
        Ok(id)
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L508)

(1) Empty positions
This allows anyone to create empty positions with no consequences at a very low cost. 

(2) Risk of `next_position_id` overflow wrapping exploits
`mint_position_B_C5_B086_D` uses `+` (`add`) to increment `next_position_id`. `add` is from ruint.1.12.3 which uses `wrapping_add` under the hood.
```rust
///.../.cargo/registry/src/index.crates.io-6f17d22bba15001f/ruint-1.12.3/src/add.rs
impl_bin_op!(Add, add, AddAssign, add_assign, wrapping_add);
```

This means when enough real positions and spam positions are created, it
s possible to spam `mint_position_B_C5_B086_D` further to cause `next_position_id` overflow wrapping to a real position with liquidity. 

This allows the exploit to be profitable.
Suppose an attack spam `mint_position_B_C5_B086_D` further causes `next_position_id` overflow wrap to 0. Now the attacker steals position 0 and becomes the owner.
If positionId 0 has liquidity worth more than the cost of the attack. The attacker can withdraw all liquidity from position id 0 and materialize a profit.

Recommendations:
Consider enforcing atomic position liquidity by adding in mint_position and enforcing a minimum liquidity threshold. 






