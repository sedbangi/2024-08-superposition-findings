# Low Risk Issues
----------------------------------


## [LOW-1] Use of unsafe math in the swap() function:
In the ```swap()``` function in pool.rs, at LOC-439 and 443, unsafe math is used:

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/pool.rs#L439

```rust
            match exact_in {
                true => {
                    state.amount_remaining -=
                        I256::unchecked_from(step_amount_in + step_fee_amount);
                    state.amount_calculated -= I256::unchecked_from(step_amount_out);
                }
                false => {
                    state.amount_remaining += I256::unchecked_from(step_amount_out);
                    state.amount_calculated +=
                        I256::unchecked_from(step_amount_in + step_fee_amount);
                }
            }
```

For these specific calculations, uniswap v3 uses safe math functions such as add() and sub().

### Recommended Mitigation Steps:
It is recommended to use safe math functions for the 2 calculations above similar to uniswap v3.


## [LOW-2] No check for zero amount in swap() function:
A user can send zero amount to the swap() function in pool.rs. Uniswap v3 has a check to prevent this However, there is no check in the swap() of pool.rs:

```rust
    pub fn swap(
        &mut self,
        zero_for_one: bool,
        amount: I256,
        mut price_limit: U256,
    ) -> Result<(I256, I256, i32), Revert> {
    .
    .
    .
    }
```

### Recommended Mitigation Steps:
Add a check to make sure ```amount``` param is not zero.


## [LOW-3] No function to set the ```fee_protocol``` of a pool:
Uniswap v3 has a function that let's the factory owner to set the protocol's fee denominator value for a pool. However, no such function exists in pool.rs. However, the pool storage contains the fee_protocol var:

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/pool.rs#L31

```rust
fee_protocol: StorageU8,
```

Therefore, the protocol won't be able to collect fee on the swap operations that would take place.

### Recommended Mitigation Steps:
Add a function that would let the admin set the fee_protocol variable.



## [LOW-4] No use of seconds and tick_cumulative values of a tick:
The tick.rs contains the following in its storage:

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/tick.rs#L54

```rust
    tick_cumulative_outside: StorageI64,
    seconds_per_liquidity_outside: StorageU160,
    seconds_outside: StorageU32,
```

However, these values are never set and used by the protocol. These values are basically maintained by uniswap v3 for other external contracts to use.

### Recommended Mitigation Steps:
It is recommended to remove these storage vars if there are not to be used.



## [LOW-5] Removal of liquidity also requires the pool to be enabled:
In LOC-687, lib.rs, it is said that liquidity removal from a position does not require the pool to be enabled:

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/lib.rs#L687

```
    /// Requires token approvals to be set if adding liquidity. Requires the caller to be the
    /// position owner. Requires the pool to be enabled unless removing liquidity.
```

However, the function ```update_position()``` in pool.rs, which is called by ```update_position_internal()``` in lib.rs requires the pool to be enabled:

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/pool.rs#L92

```rust
assert_or!(self.enabled.get(), Error::PoolDisabled);
```

### Recommended Mitigation Steps:
Consider allowing removal of liquidity even when the pool is not enabled, that is, when delta is negative
