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

### Low-04 SeawaterAMM proxy has immutable facet function signatures, which might cause conflict in future upgrades.
**Instances(38)**
SeawaterAMM.sol has wrapper function declarations for functions in implementation facets. This makes these function signatures in all wrapped implementation contract methods immutable. 

When a future implementation contract needs to change a function signature, the wrapper declarations in effectively obsolete. These immutable wrappers are not necessary and might be in conflict with future upgrades.

For example, 
```solidity
//pkg/sol/SeawaterAMM.sol
    function swap2ExactInPermit236B2FDD8(
        address /* from */,
        address /* to */,
        uint256 /* amount */,
        uint256 /* minOut */,
        uint256 /* nonce */,
        uint256 /* deadline */,
        bytes memory /* sig */
    ) external returns (uint256, uint256) {
        directDelegate(_getExecutorSwapPermit2());
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L249)

Recommendations:
Consider only using fallback to delegate calls.

### Low-05 Facet contracts cannot be upgraded independently from other facet contracts
**Instances(1)**
SeawaterAMM.sol is intended to work with multiple facet contracts. The issue is there is no method to update one facet contract without re-writing other facet contract addresses simultaneously.

When one facet address needs to be updated, all facet addresses have to be re-written in storage. This is wasteful.
```solidity
//pkg/sol/SeawaterAMM.sol
    function updateExecutors(
        ISeawaterExecutorSwap executorSwap,
        ISeawaterExecutorSwapPermit2 executorSwapPermit2,
        ISeawaterExecutorQuote executorQuote,
        ISeawaterExecutorPosition executorPosition,
        ISeawaterExecutorUpdatePosition executorUpdatePosition,
        ISeawaterExecutorAdmin executorAdmin,
        ISeawaterExecutorFallback executorFallback
    ) public onlyProxyAdmin {
        _setProxies(
            executorSwap,
            executorSwapPermit2,
            executorQuote,
            executorPosition,
            executorUpdatePosition,
            executorAdmin,
            executorFallback
        );
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L115)

Recommendations:
Consider allowing updating one executor independently when necessary.

### Low-06 `swap_2` allows 0 amount swap to pass
**Instances(1)**
`swap_2` will not revert when user input 0 amount. 
```rust
//pkg/seawater/src/lib.rs
    fn swap_2_internal(
        pools: &mut Pools,
        from: Address,
        to: Address,
        amount: U256,
        min_out: U256,
    ) -> Result<(U256, U256, U256, I256, i32, i32), Revert> {
...
        let (amount_in, interim_usdc_out, final_tick_in) = pools.pools.setter(from).swap(
            true,
            amount,
            // swap with no price limit, since we use min_out instead
            tick_math::MIN_SQRT_RATIO + U256::one(),
        )?;
...
        let (amount_out, interim_usdc_in, final_tick_out) = pools.pools.setter(to).swap(
            false,
            interim_usdc_out,
            tick_math::MAX_SQRT_RATIO - U256::one(),
        )?;
        let amount_in = amount_in.abs_pos()?;
        let amount_out = amount_out.abs_neg()?;
...
        assert_eq_or!(interim_usdc_out, interim_usdc_in, Error::InterimSwapNotEq);
        assert_or!(amount_out >= min_out, Error::MinOutNotReached);
        Ok((
            original_amount,
            amount_in,
            amount_out,
            interim_usdc_out,
            final_tick_in,
            final_tick_out,
        ))
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L241-L242)

Note that pool::swap will simply skip while loop due to [state.amount_remaining will be 0](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L378C16-L378C38).

Recommendations:
Consider check and revert when the amount input is 0 to prevent no-op.

### Low-07 Change `lower` into `upper`.
**Instances(1)**
In `position_tick_upper_67_F_D_55_B_A`, there is a typo. When querying the upper tick, the variable assigned is `lower`, which reduces code readability.
```rust
//pkg/seawater/src/lib.rs
    pub fn position_tick_upper_67_F_D_55_B_A(
        &self,
        pool: Address,
        id: U256,
    ) -> Result<i32, Revert> {
|>      let lower = self.pools.getter(pool).get_position_tick_upper(id);
        Ok(lower.sys())
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L606)

Recommendations:
Change `lower` into `upper`.

### Low-08 Position can be updated in the wrong pool due to insufficient checks.
**Instances(3)**
Each position is associated with the pool(token0) where the position info (StoragePool::positions::StoragePositionInfo). 

The issue is update_position() / incr_position() / decr_position() has insufficient checks to ensure the position info updated is associated with the correct pool(token0). As a result, position info (`StoragePositionInfo`) can be updated in the wrong pool.

Take `update_position_C_7_F_1_F_740()` -> `update_position_internal()` as an example. We see that there is no check that the input `pool: Address` is the pool(token0) that the position is associated with. In addition, in StoragePool::positions, `StoragePositionInfo` is stored in a key-value(id ->info) mapping, which means even if positionId doesn't exist in the pool,  `StoragePositionInfo` will be default value.
```rust
    pub fn update_position_internal(
        &mut self,
        pool: Address,
        id: U256,
        delta: i128,
        permit2: Option<(Permit2Args, Permit2Args)>,
    ) -> Result<(I256, I256), Revert> {
        assert_eq_or!(
            msg::sender(),
            self.position_owners.get(id),
            Error::PositionOwnerOnly
        );
|>      let (token_0, token_1) = self.pools.setter(pool).update_position(id, delta)?;
...
//pkg/seawater/src/pool.rs
    pub fn update_position(&mut self, id: U256, delta: i128) -> Result<(I256, I256), Revert> {
        // the pool must be enabled
        assert_or!(self.enabled.get(), Error::PoolDisabled);

|>        let position = self.positions.positions.get(id);
...
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L94)

See added unit test `position_with_incorrect_pool_address()` below. 
POC: a user mint position in pool0(token0) then update position liquidity with pool1(token1). The pool1's liquidity is updated instend of pool0.
```rust
//pkg/seawater/tests/lib.rs
#[test]
fn position_with_incorrect_pool_address() {
    test_utils::with_storage::<_, Pools, _>(
        Some(address!("3f1Eae7D46d88F08fc2F8ed27FCb2AB183EB2d0E").into_array()), // sender
        None,
        None,
        None,
        |contract| -> Result<(), Vec<u8>> {
            let token0 = address!("9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0");
            let token1 = address!("9fE46736679d2D9a65F0992F2272dE9f3c7fa6e1");
            contract
                .ctor(msg::sender(), Address::ZERO, Address::ZERO)
                .unwrap();
            contract
                .create_pool_D650_E2_D0(
                    token0,
                    U256::from_limbs([0, 42949672960, 0, 0]), //792281625142643375935439503360
                    500,                                      // fee
                    10,                                       // tick spacing
                    u128::MAX,
                )
                .unwrap();
            contract
                .create_pool_D650_E2_D0(
                    token1,
                    U256::from_limbs([0, 42949672960, 0, 0]), //792281625142643375935439503360
                    500,                                      // fee
                    10,                                       // tick spacing
                    u128::MAX,
                )
                .unwrap();
            contract.enable_pool_579_D_A658(token0, true).unwrap();
            contract.enable_pool_579_D_A658(token1, true).unwrap();
            //mint position in token0 pool
            contract
                .mint_position_B_C5_B086_D(token0, 39120, 50100)
                .unwrap();
            let id = U256::ZERO;
            let (amount_0_in, amount_1_in) = contract
                .update_position_C_7_F_1_F_740(token1, id, 2000)
                .unwrap();
            println!("amount_0_in: {amount_0_in}; amount_1_in: {amount_1_in}");
            //liquidity is updated in token1 pool instead!
            let liquidity = contract.position_liquidity_8_D11_C045(token1, id).unwrap();
            println!("token1 pool's position id liquidity: {liquidity}");
            Ok(())
        },
    )
    .unwrap()
}
```
Test results:
```bash
     Running tests/lib.rs (target/debug/deps/lib-b27e97df8f1d4fcd)

running 1 test
amount_0_in: 0; amount_1_in: 0
token1 pool's position id liquidity: 2000
test position_with_incorrect_pool_address ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 21 filtered out; finished in 0.00s
```
Recommendations:
In update_position(), incr_position(), decr_position() add check that the position info returned from storagePool’s position’s lower and upper tick are initialized (non-zero).

### Low-09 Pools that have not been created can be enabled. Users can interact with a pool in an uninitialized state.
**Instances(1)**
When enabling a pool, enable_pool_579_D_A658() doesn’t check whether the pool has been created (storagePool.sqrt_price ≠ 0).
In liquidity/swap operations, only the pool's state of whether it is enabled is checked. As a result a user can interact with a pool in its raw state which leads to unexpected state change due to pool's price/tick is not set and will start from 0.
```rust
//pkg/seawater/src/lib.rs
    pub fn enable_pool_579_D_A658(&mut self, pool: Address, enabled: bool) -> Result<(), Revert> {
            assert_or!(
            self.seawater_admin.get() == msg::sender()
                || self.emergency_council.get() == msg::sender()
                || self.authorised_enablers.get(msg::sender()),
            Error::SeawaterAdminOnly
        );

        if self.emergency_council.get() == msg::sender()
            && self.seawater_admin.get() != msg::sender()
            && enabled
        {
            // Emergency council can only disable!
            return Err(Error::SeawaterEmergencyOnlyDisable.into());
        }

        self.pools.setter(pool).set_enabled(enabled);
        Ok(())
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L1185)
Recommendations:
Consider adding a check to ensure pool’s current sqrt_price ≠0 (initialized).

### Low-10  Unnecessary implementation -send_amounts_from_sender()
**Instances(1)**
send_amounts_from_sender simply wraps erc20 safeTransferFrom and will transfer token from msg.sender to recipient_addrs.

This is no difference than msg.sender directly calling erc20's safeTransferFrom. The call doesn't have to route through SeawaterAMM.
```rust
//pkg/seawater/src/lib.rs
    pub fn send_amounts_from_sender(
        &mut self,
        token: Address,
        recipient_addrs: Vec<Address>,
        recipient_amounts: Vec<U256>,
    ) -> Result<(), Revert> {
        assert_eq_or!(
            msg::sender(),
            self.seawater_admin.get(),
            Error::SeawaterAdminOnly
        );

        for (addr, amount) in recipient_addrs.iter().zip(recipient_amounts.iter()) {
            erc20::take_from_to(token, *addr, *amount)?;
        }

        Ok(())
    }
}
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L1249)
```rust
//pkg/seawater/src/wasm_erc20.rs
pub fn take_from_to(token: Address, recipient: Address, amount: U256) -> Result<(), Error> {
    safe_transfer_from(token, msg::sender(), recipient, amount)
}
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/wasm_erc20.rs#L167)

Recommendations:
Consider removing unnecessary functions.

### Low-11 OnwershipNFTs's approval mechanism is not compliant with eip-721 standard. 
**Instances(1)**
eip-721 allows an authorized operator to set approval address for compatibility reasons. Current implementation only allows the tokenId owner to set token approval, not an authorized operator (`isApprovedForAll`).

>Additionally, an authorized operator may set the approved address for an NFT. This provides a powerful set of tools for wallet, broker and auction applications to quickly use a large number of NFTs.

See ERC-721 [here](https://eips.ethereum.org/EIPS/eip-721#specification)

We see current approve mechanism only allows the tokenId owner to set single token approval due to requiring ownerOf(_tokenId) == msg.sender.
```solidity
//pkg/sol/OwnershipNFTs.sol
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }

    function _requireAuthorised(address _from, uint256 _tokenId) internal view {
        // revert if the sender is not authorised or the owner
        bool isAllowed = msg.sender == _from ||
            isApprovedForAll[_from][msg.sender] ||
            msg.sender == getApproved[_tokenId];
        require(isAllowed, "not allowed");
        require(ownerOf(_tokenId) == _from, "_from is not the owner!");
    }
```
Recommendations:
Consider allowing an authorized operator to set approved address per eip-721.

### Low-12 OwnershipNfts.sol is missing ERC165::supportInterface, no incompliance with eip-721.
**Instances(1)**
According to eip-721, erc721 contract should expose its interface based on ERC165.

>Every ERC-721 compliant contract must implement the ERC721 and ERC165 interfaces

See [eip-721](https://eips.ethereum.org/EIPS/eip-721#specification).

OwnershipNfts.sol is missing ERC165::supportInterface implementation.

Recommendations:
Add supportsInterface method per eip-721.

### Low-13 `swap_internal` has invalid check on amount_0 and amount_1
**Instances(1)**
`swap_internal` checks the returned amount_0 and amount_1 (token in / token out amounts) from storagePool::swap. But the checks allow an edge case where either amount_0_abs is 0 or amount_1_abs is 0, which shouldn’t be a valid state in normal swap cases both token in and token out amount should be greater than zero.

```rust
//pkg/seawater/src/lib.rs
    pub fn swap_internal(
    ...
            assert_or!(
            amount_0_abs > U256::zero() || amount_1_abs > U256::zero(),
            Error::SwapResultTooLow
        );
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L179-L181)

Recommendations:
Consider changing the check into
```rust
        assert_or!(
            amount_0_abs > U256::zero() && amount_1_abs > U256::zero(),
            Error::SwapResultTooLow
        );
```


### Low-14 NFT Position minted without checking without checking user implements onERC721Received. 
**Instances(1)**
We see in `mint_position_B_C5_B086_D()` -> `grant_position` there is no onERC721Received call to the owner.
```rust
//pkg/seawater/src/lib.rs
    pub fn mint_position_B_C5_B086_D(
        &mut self,
        pool: Address,
        lower: i32,
        upper: i32,
    ) -> Result<U256, Revert> {
    ...
            self.grant_position(owner, id);
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L508)
```rust
    fn grant_position(&mut self, owner: Address, id: U256) {
        // set owner
        self.position_owners.setter(id).set(owner);

        // increment count
        let owned_positions_count = self.owned_positions.get(owner) + U256::one();
        self.owned_positions
            .setter(owner)
            .set(owned_positions_count);
    }
```
(https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L463)

Recommendations:
In mint_position flow, add a call to msg.sender/owner to ensure if the owner is a contract, it implements onERC721Received.
