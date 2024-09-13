| Issue No. | Issue Title                                                    |
|-----------|---------------------------------------------------------------|
| [01]      | The swap function does not check whether the amount is zero   |
| [02]      | The decrPosition09293696 function is missing a parameter      |
| [03]      | The quote2CD06B86E function has misleading parameter comments |
| [04]      | The position_tick_upper_67_F_D_55_B_A function contains misleading variable names |
| [05]      | The transfer_position_E_E_C7_A3_C_D function has incorrect descriptions in its Calling Requirements comments |
| [06]      | Single-token approvals should not be able to call approve     |


## [01] The swap function does not check whether the amount is zero
### Link
github:[https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L303](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L303)

### Description
In the swap function, swaps with an amount of zero should be rejected to prevent invalid calls.  
```rust
    pub fn swap(
        &mut self,
        zero_for_one: bool,
        amount: I256,
        mut price_limit: U256,
    ) -> Result<(I256, I256, i32), Revert> {
        assert_or!(self.enabled.get(), Error::PoolDisabled);
```
### Recommendation
Check whether the swap amount is zero
```diff
    pub fn swap(
        &mut self,
        zero_for_one: bool,
        amount: I256,
        mut price_limit: U256,
    ) -> Result<(I256, I256, i32), Revert> {
        assert_or!(self.enabled.get(), Error::PoolDisabled);
+       assert_or!(amount > 0, Error::InvalidAmount);
```

## [02] The decrPosition09293696 function is missing a parameter.
### Link
github:[https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L476](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L476)
### Description
The `decrPosition09293696` function is intended to call the `decr_position_09293696` function in the Rust contract, but it is missing the `pool` parameter.  

```solidity
    function decrPosition09293696(
        uint256 /* id */,
        uint256 /* amount0Min */,
        uint256 /* amount1Min */,
        uint256 /* amount0Max */,
        uint256 /* amount1Max */
    ) external returns (uint256, uint256) {
        directDelegate(_getExecutorUpdatePosition());
    }
```
```rust
    /// Refreshes and updates liquidity in a position, transferring tokens to the user with restrictions.
    /// See [Self::adjust_position_internal].
    #[allow(non_snake_case)]
    pub fn decr_position_09293696(
        &mut self,
        pool: Address,
        id: U256,
        amount_0_min: U256,
        amount_1_min: U256,
        amount_0_max: U256,
        amount_1_max: U256,
    ) -> Result<(U256, U256), Revert> {
        self.adjust_position_internal(
            pool,
            id,
            amount_0_min,
            amount_1_min,
            amount_0_max,
            amount_1_max,
            true,
            None,
        )
    }
}
```
### Recommendation
Add the `pool` parameter to the decrPosition09293696 function

```diff
    function decrPosition09293696(
+       address /* pool */,
        uint256 /* id */,
        uint256 /* amount0Min */,
        uint256 /* amount1Min */,
        uint256 /* amount0Max */,
        uint256 /* amount1Max */
    ) external returns (uint256, uint256) {
        directDelegate(_getExecutorUpdatePosition());
    }
```

## [03] The `quote2CD06B86E` function has misleading parameter comments.
### Link
github: [https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L225](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L225)  
### Description
The `quote2CD06B86E` function is a proxy call to the `quote_2_C_D06_B86_E` function in the Rust contract, but its parameter comments contain incorrect variable names, which could lead to confusion for users.  
```solidity
    /// @inheritdoc ISeawaterExecutorQuote
    function quote2CD06B86E(address /* to */, address /* from */, uint256 /* amount */, uint256 /* minOut*/) external {
        directDelegate(_getExecutorQuote());
    }
```
```rust
    #[allow(non_snake_case)]
    pub fn quote_2_C_D06_B86_E(
        &mut self,
        from: Address,
        to: Address,
        amount: U256,
        min_out: U256,
    ) -> Result<(), Revert> {
        ...
    }
```
It can be seen that the comments for the `from` and `to` parameters are reversed.  
### Recommendation
```diff
    /// @inheritdoc ISeawaterExecutorQuote
-   function quote2CD06B86E(address /* to */, address /* from */, uint256 /* amount */, uint256 /* minOut*/) external {
+   function quote2CD06B86E(address /* from */, address /* to */, uint256 /* amount */, uint256 /* minOut*/) external {
        directDelegate(_getExecutorQuote());
    }
```

## [04] The `position_tick_upper_67_F_D_55_B_A` function contains misleading variable names.
### Link
github:[https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L606](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L606)
### Description
The `position_tick_upper_67_F_D_55_B_A` function is used to retrieve the `tick_upper` of a position, but it uses the name `lower` for a temporary variable when storing the tick. This naming inconsistency can mislead developers.
```rust
    #[allow(non_snake_case)]
    pub fn position_tick_upper_67_F_D_55_B_A(
        &self,
        pool: Address,
        id: U256,
    ) -> Result<i32, Revert> {
        let lower = self.pools.getter(pool).get_position_tick_upper(id);
        Ok(lower.sys())
    }
```
### Recommendation
```diff
    #[allow(non_snake_case)]
    pub fn position_tick_upper_67_F_D_55_B_A(
        &self,
        pool: Address,
        id: U256,
    ) -> Result<i32, Revert> {
-       let lower = self.pools.getter(pool).get_position_tick_upper(id);
+       let upper = self.pools.getter(pool).get_position_tick_upper(id);
-       Ok(lower.sys())
+       Ok(upper.sys())
    }
```

## [05] The `transfer_position_E_E_C7_A3_C_D` function has incorrect descriptions in its Calling Requirements comments.
### Link
github:[https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L549](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L549)
### Description
The `transfer_position_E_E_C7_A3_C_D` function restricts calls to only the `nft_manager`, but the Calling Requirements comment incorrectly states, "Requires that the `from` address is the current owner of the position." This incorrect description could mislead users.
```rust
    /// Transfers a position's ownership from one address to another. Only usable by the NFT
    /// manager account.
    ///
    /// # Calling requirements
    /// Requires that the `from` address is the current owner of the position.
    ///
    /// # Errors
    /// Requires the caller be the NFT manager.
    #[allow(non_snake_case)]
    pub fn transfer_position_E_E_C7_A3_C_D(
        &mut self,
        id: U256,
        from: Address,
        to: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(msg::sender(), self.nft_manager.get(), Error::NftManagerOnly);
        ...
    }
```
### Recommendation
```diff
    /// Transfers a position's ownership from one address to another. Only usable by the NFT
    /// manager account.
    ///
    /// # Calling requirements
-   /// Requires that the `from` address is the current owner of the position.
+   /// Requires that the `from` address is the nft_manager.
    ///
    /// # Errors
    /// Requires the caller be the NFT manager.
```

## [06] single-token approvals should not be able to call approve
### Link
github:[https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L161](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L161)  
### Description
The `approve` function is used for authorization requests on a single position. In OpenZeppelin's ERC721 implementation, single-token approvals should not be able to use `approve` to transfer their token permissions to others. However, in the OwnershipNFTs implementation, this kind of call is allowed.  
OwnershipNFTs:
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
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```
OpenZeppelin:
```solidity
    function _approve(address to, uint256 tokenId, address auth, bool emitEvent) internal virtual {
        // Avoid reading the owner unless necessary
        if (emitEvent || auth != address(0)) {
            address owner = _requireOwned(tokenId);

            // We do not use _isAuthorized because single-token approvals should not be able to call approve
            if (auth != address(0) && owner != auth && !isApprovedForAll(owner, auth)) {
                revert ERC721InvalidApprover(auth);
            }

            if (emitEvent) {
                emit Approval(owner, to, tokenId);
            }
        }

        _tokenApprovals[tokenId] = to;
    }
```
### Recommendation
```diff
    function approve(address _approved, uint256 _tokenId) external payable {
-       _requireAuthorised(msg.sender, _tokenId);
+       require(msg.sender == _from || isApprovedForAll[_from][msg.sender], "not allowed");
+       require(ownerOf(_tokenId) == _from, "_from is not the owner!");
        getApproved[_tokenId] = _approved;
    }
```
