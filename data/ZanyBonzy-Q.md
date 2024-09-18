### 1. OwnershipNFTs.sol doesn't fully match EIP-721's implementation.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L182

#### Impact

`tokenURI` doesn't enforce that the token being checked is a valid token, which can lead to malicious users impersonating bogus NFTs as the ownership NFT. Unsuspecting users would attempt to query the token uri and would be returned with a valid uri, since the function doesn't ensure that the tokenId is a valid NFT.

Also, according to EIP-721 metadata standard, the function should revert if `_tokenId` is not a valid NFT.

> /// @notice A distinct Uniform Resource Identifier (URI) for a given asset.
>    /// @dev Throws if `_tokenId` is not a valid NFT. URIs are defined in RFC

```solidity
    function tokenURI(uint256 /* _tokenId */) external view returns (string memory) {
        return TOKEN_URI;
    }
```

#### Recommended Mitigation Steps

Add the check for NFT existence in the function by checking that the owner is not address(0)

***
### 2. OwnershipNFTs's approve function allows approved users to also perform approvals.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160-L163

#### Impact

OwnershipNFTs's approve function allows approved users to also perform approvals, which can lead to approval loss.

The approve function checks if `msg.sender` is authorized. 

```solidity
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```
This is enforced by the `_requireAuthorised` which ensures that caller is either owner, owner's operator or approved to spend the token.
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
This is an issue because any user that had be approved to spend the token risks losing their approval after calling the `approve` function.

#### Recommended Mitigation Steps

Recommend limiting the approval to the owner and operator instead.
***

### 3. Exisitng backdoor for owner/operator in the `approve` function

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160-L163

#### Impact

The approve function allows operator and owner to approve themselves to spend the token. This can be used to give themselves a backdoor to the toeken. If for example the operator's status is to be revoked, he can always call the `approve` function on the token, to approve himself to spend the token. So even if his operator status is revoked, he still has access to the tokens.

```solidity
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```

#### Recommended Mitigation Steps

Ensure that the owner/operator cannot approve themselves to spend a token.

***

### 4. Consider automatically collecting fees before transferring positons

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L554-L569

#### Impact

`transfer_position_E_E_C7_A3_C_D` directly removes the positions from the sender and adds it to the receiver without making considerations for potential fees that the position has accrued. 

```rust
    pub fn transfer_position_E_E_C7_A3_C_D(
        &mut self,
        id: U256,
        from: Address,
        to: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(msg::sender(), self.nft_manager.get(), Error::NftManagerOnly);

        self.remove_position(from, id);
        self.grant_position(to, id);

        #[cfg(feature = "log-events")]
        evm::log(events::TransferPosition { from, to, id });

        Ok(())
    }
```

#### Recommended Mitigation Steps

Recommend introducting `collect_single_to_6_D_76575_F` into the `transfer_position_E_E_C7_A3_C_D` function to help users directly claim fees before transferring their positions.

***


### 5. Functions to claim fees should be accessible even when pool is disabled

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L562-L565

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L538-L559

#### Impact

The `collect` and `collect_protocol` functions allow users and protocol admins to claim their fees. These functions are however not accessible when the pool is disabled. As a rule of thumb, functions to remove rewards or withdraw tokens from a protocol should always be accessible to help users claim resuce their tokens. Also a malicious admin can disable the pool and renounce his ownership, permanently locking the tokens in the protocol.

```rust
    pub fn collect(&mut self, id: U256) -> Result<(u128, u128), Revert> {
>>      assert_or!(self.enabled.get(), Error::PoolDisabled);
        Ok(self.positions.collect_fees(id))
    }
```


```rust
    pub fn collect_protocol(
        &mut self,
        amount_0: u128,
        amount_1: u128,
    ) -> Result<(u128, u128), Revert> {
>>      assert_or!(self.enabled.get(), Error::PoolDisabled);

        let owed_0 = self.protocol_fee_0.get().sys();
        let owed_1 = self.protocol_fee_1.get().sys();

        let amount_0 = u128::min(amount_0, owed_0);
        let amount_1 = u128::min(amount_1, owed_1);

        if amount_0 > 0 {
            self.protocol_fee_0.set(U128::lib(&(owed_0 - amount_0)));
        }
        if amount_1 > 0 {
            self.protocol_fee_1.set(U128::lib(&(owed_1 - amount_1)));
        }

        Ok((amount_0, amount_1))
    }
```


#### Recommended Mitigation Steps

Recommend removing the check for pool activity in the functions.

***


### 6. Certain getter functions limited to admin executor.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L400-L435

#### Impact

The below functions getter functions, allowing to retrieve various pool information but are limited to the executor admin. As a result, interactions with external protocols or just ordinary users trying to query these function will fail. Not sure the best severity for this as it affects external integrations.

```solidity
    function sqrtPriceX967B8F5FC5(address /* pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feesOwed22F28DBD(
        address /* pool */,
        uint256 /* position */
    ) external returns (uint128, uint128) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function curTick181C6FD9(address /* pool */) external returns (int32) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function tickSpacing653FE28F(address /* pool */) external returns (uint8) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeBB3CF608(address /* pool */) external returns (uint32) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeGrowthGlobal038B5665B(address /* pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeGrowthGlobal1A33A5A1B(address /*pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }
```

#### Recommended Mitigation Steps
Recommend limiting to `_getExecutorPosition` instead.

***

### 7. `enablePool579DA658` can only be queried by admin, which locks out authorized enablers and emergency council

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L181-L187

#### Impact

The function to enable pool - `enablePool579DA658` is intended to be called by the admin, emergency council and authorized enablers. But the function in SeawaterAMM.sol, only the admin can call the function locking out the other privileged roles.

```solidity
    function enablePool579DA658(
        address /* pool */,
        bool /* enabled */
    ) external {
        directDelegate(_getExecutorAdmin());
    }
```

***

### 8. Non-existent function in referenced on SeawaterAMM.sol
 
* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L487-L502

#### Impact

The `incrPositionPermit25468326E` function is implemented in SeawaterAMM.sol, but isn't implemented in any of the pool's backend library. As a result any calls to the function will always fail.

```solidity
    function incrPositionPermit25468326E(
        address /* token */,
        uint256 /* id */,
        uint256 /* amount0Min */,
        uint256 /* amount1Min */,
        uint256 /* nonce0 */,
        uint256 /* deadline0 */,
        uint256 /* amount0Max */,
        bytes memory /* sig0 */,
        uint256 /* nonce1 */,
        uint256 /* deadline1 */,
        uint256 /* amount1Max */,
        bytes memory /* sig1 */
    ) external returns (uint256, uint256) {
        directDelegate(_getExecutorUpdatePosition());
    }
```

#### Recommended Mitigation Steps

Recommend removing the unimplemented function.
***

### 9. OwnershipNFTs.sol holds payable functions but have no way to rescue 

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L118-L162


#### Impact

A number of functions in OwnershipNFTs.sol are marked as payable, meaning they can receive ETH. But the contract implementation doesn't make use of ETH, nor is there any function to recover any ETH sent. As a result the tokens will be lost forever.

```solidity
    function transferFrom(
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata /* _data */
    ) external payable {
        // checks that the user is authorised
        _transfer(_from, _to, _tokenId);
    }

    /// @inheritdoc IERC721Metadata
    function transferFrom(
        address _from,
        address _to,
        uint256 _tokenId
    ) external payable {
        // checks that the user is authorised
        _transfer(_from, _to, _tokenId);
    }

    /// @inheritdoc IERC721Metadata
    function safeTransferFrom(
        address _from,
        address _to,
        uint256 _tokenId
    ) external payable {
        _transfer(_from, _to, _tokenId);
        _onTransferReceived(msg.sender, _from, _to, _tokenId);
    }

    /// @inheritdoc IERC721Metadata
    function safeTransferFrom(
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata /* _data */
    ) external payable {
        _transfer(_from, _to, _tokenId);
        _onTransferReceived(msg.sender, _from, _to, _tokenId);
    }

    /// @inheritdoc IERC721Metadata
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```
#### Recommended Mitigation Steps

Recommend removing the payable modifier from the listed functions, or introducing a function to recover andy excess tokens in the contract.
***

### 10. Function to directly adjust position is not exposed in SeawaterAMM.sol

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L730

#### Impact

lib.rs holds the `adjust_position_internal` which allows users to change their position details according to sepcified giving bool. However, this function is not exposed in SeawaterAMM.sol, so as such cannot be directly queried. Also, since there's an issue with the `decrPosition09293696` function (an incorrect function definition), there's no way to for users to reduce their positons.

#### Recommended Mitigation Steps

Recommend exposing the function.
***

### 11. Potential fix for `eli_incr_position` test issue.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/maths/sqrt_price_math.rs#L252-L267

https://github.com/Uniswap/v3-periphery/blob/b325bb0905d922ae61fcc7df85ee802e8df5e96c/contracts/libraries/LiquidityAmounts.sol#L128

#### Impact

When looking the `get_amounts_for_delta` function, it can be observed that its very similar to the `getAmountsForLiquidity` function in Uniswap's LiquidityAmounts.sol. A number of the contracts' functions are very similar. However, in this case, `get_amounts_for_delta` queries `get_amount_0_delta` and `get_amount_1_delta` rather than `get_liquidity_for_amount_0` and `get_liquidity_for_amount_0` which could be a reason for the failure.

```rust
pub fn get_amounts_for_delta(
    sqrt_ratio_x_96: U256,
    mut sqrt_ratio_a_x_96: U256,
    mut sqrt_ratio_b_x_96: U256,
    liquidity: i128,
) -> Result<(I256, I256), Error> {
    if sqrt_ratio_a_x_96 > sqrt_ratio_b_x_96 {
        (sqrt_ratio_a_x_96, sqrt_ratio_b_x_96) = (sqrt_ratio_b_x_96, sqrt_ratio_a_x_96)
    };
    Ok(if sqrt_ratio_x_96 <= sqrt_ratio_a_x_96 {
        (
            get_amount_0_delta(sqrt_ratio_a_x_96, sqrt_ratio_b_x_96, liquidity)?,
            I256::ZERO,
        )
    } else if sqrt_ratio_x_96 < sqrt_ratio_b_x_96 {
        (
            get_amount_0_delta(sqrt_ratio_x_96, sqrt_ratio_b_x_96, liquidity)?,
            get_amount_1_delta(sqrt_ratio_a_x_96, sqrt_ratio_x_96, liquidity)?,
        )
    } else {
        (
            I256::ZERO,
            get_amount_1_delta(sqrt_ratio_a_x_96, sqrt_ratio_b_x_96, liquidity)?,
        )
    })
}
```

***


### 12. Functions involving permit can easily be dossed

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L322

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L281

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L249

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L230

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L487

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L408

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L439


#### Impact

The `swap_permit_2_E_E84_A_D91`, `swap_2_exact_in_permit_2_36_B2_F_D_D8`, `incrPositionPermit25468326E`, `swapOutPermit23273373B`, `swapOut5E08A399`, `swapInPermit2CEAAB576`, `swap2ExactInPermit236B2FDD8` etc functions allow off-chain signed approvals to be used on-chain, saving gas and improving user experience.

However, including these functions will make the transaction susceptible to DOS via frontrunning. An attacker can observe the transaction in the mempool, extract the permit signature and values from the calldata and execute the permit before the original transaction is processed. This would consume the nonce associated with the user's permit and cause the original transaction to fail due to the now-invalid nonce.

This attack vector has been previously described in Permission Denied - [The Story of an EIP that Sinned](https://www.trust-security.xyz/post/permission-denied).

#### Recommended Mitigation Steps

Recommend wrapping the permit functionalities in a try catch block or its rust equivalent. Something like this may work.

```solidity
try IERC20Permit(token).permit(msgSender, address(this), value, deadline, v, r, s) {
    // Permit executed successfully, proceed
} catch {
    // Check allowance to see if permit was already executed
    uint256 currentAllowance = IERC20(token).allowance(msgSender, address(this));
    if(currentAllowance < value) {
        revert("Permit failed and allowance insufficient");
    }
    // Otherwise, proceed as if permit was successful
}
```
***

### 13. Lack of a deadline parameter during interactions with pools

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L315

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L327

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L408

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L439

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L230

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L249

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L262

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L281

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L304

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L322

#### Impact

Functions `swap_904369_B_E`, `swap_2_exact_in_41203_F1_D`, `swap_permit_2_E_E84_A_D91`, `swap_2_exact_in_permit_2_36_B2_F_D_D8`, `swapOutPermit23273373B`, `swapOut5E08A399`, `swapIn32502CA71`, `swapInPermit2CEAAB576` etc are missing deadline check.

As front-running is a key aspect of AMM design, the deadline is a useful tool to ensure that users' transactions cannot be "saved for later" by miners or stay longer than needed in the mempool. The longer transactions stay in the mempool, the more likely it is that MEV bots can steal positive slippage from the transaction.

#### Recommended Mitigation Steps

Consider introducing a deadline parameter in all the pointed-out functions.

***

### 14. Ownership NFts lack the mint and burn functionalities

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L13

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L495-L544

#### Impact

Not sure the best severity for this, as the contract is described as an interface for tracking ownership positions. Yet at the same time, it holds functionalities that are pertinent to real NFTs, it can be approved and transfered. In any case, the contract lacks any functionality to mint and burn the tokens. As such, these tokens can't really be minted or burnt from users. 

Also, when positions are minted/burned from the pool, no function is called to mint/burn NFTs from the caller.

```rs
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

        self.grant_position(owner, id);

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
```rs
    #[allow(non_snake_case)]
    pub fn burn_position_AE401070(&mut self, id: U256) -> Result<(), Revert> {
        let owner = msg::sender();
        assert_eq_or!(
            self.position_owners.get(id),
            owner,
            Error::PositionOwnerOnly
        );

        self.remove_position(owner, id);

        #[cfg(feature = "log-events")]
        evm::log(events::BurnPosition { owner, id });

        Ok(())
    }
```

***


### 15. Remove debugging code from production and refurbish test suite

* Lines of code

https://github.com/search?q=repo%3Acode-423n4%2F2024-08-superposition%20%23%5Bcfg(feature%20%3D%20%22testing-dbg%22)%5D&type=code

#### Impact

Lots of the contracts in scope stll hold the debugging code (See link above). As a result, the codebase looked a bit cluttered. Also, the test suite and its configuration was very difficult to get under control. Lots of weird errors and failures.

```rs
#[cfg(feature = "testing-dbg")]
```

***

***


### 16. `calldata` is not used during NFT transfers

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L148-L157

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L118-L126

#### Impact

`transferFrom` and `safeTransferFrom` take the `calldata` parameter but its not in used, especially in the `_onTransferReceived` hook. External integrations depending on the calldata will have issues due to the parameter not being used.

```solidity
    function transferFrom(
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata /* _data */
    ) external payable {
        // checks that the user is authorised
        _transfer(_from, _to, _tokenId);
    }
```

```solidity
    function safeTransferFrom(
        address _from,
        address _to,
        uint256 _tokenId,
        bytes calldata /* _data */
    ) external payable {
        _transfer(_from, _to, _tokenId);
        _onTransferReceived(msg.sender, _from, _to, _tokenId);
    }
```

#### Recommended Mitigation Steps

Recommend implementing the use of this parameter, or removing the functions since its redundant.

***

### 17. Prevent transfer to self when transferring positions

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L553-L569

#### Impact

`transfer_position_E_E_C7_A3_C_D` allows users to transfer tokens to themselves. It should return if from == to.

```rust
    pub fn transfer_position_E_E_C7_A3_C_D(
        &mut self,
        id: U256,
        from: Address,
        to: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(msg::sender(), self.nft_manager.get(), Error::NftManagerOnly);

        self.remove_position(from, id);
        self.grant_position(to, id);

        #[cfg(feature = "log-events")]
        evm::log(events::TransferPosition { from, to, id });

        Ok(())
    }
```

***

### 18. `swap_2_exact_in_41203_F1_D` shouldn't allow swapping between same pools

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L327-L336

#### Impact

`swap_2_exact_in_41203_F1_D` allows swapping between the same pools which malicious users can attempt to leverage, trying to drain pool liqudity. Recommend reverting or returning if from == to.

```rs
    pub fn swap_2_exact_in_41203_F1_D(
        &mut self,
        from: Address,
        to: Address,
        amount: U256,
        min_out: U256,
    ) -> Result<(U256, U256), Revert> {
        Pools::swap_2_internal_erc20(self, from, to, amount, min_out, None)
    }
}
```

***
