## Summary

---

### Low Risk Issues
|Id|Title|Instances|
|:--:|:------|:--:|
|[[L-01]](#l-01-a-user-can-burn-their-position-before-the-nft-manager-transfers-it)| A user can burn their position before the nft manager transfers it | 1 |
|[[L-02]](#l-02-a-pool-can-be-re-initialized-by-setting-the-price-to-zero)| A pool can be re-initialized by setting the price to zero | 1 |
|[[L-03]](#l-03-mod-operation-doesnt-revert-on-overflow-in-release-mode)| `mod` operation doesn't revert on overflow in release mode | 1 |
|[[L-04]](#l-04-file-allows-a-version-of-solidity-that-is-susceptible-to-selector-related-optimizer-bug)| File allows a version of solidity that is susceptible to .selector-related optimizer bug | 3 |
|[[L-05]](#l-05-vulnerability-to-storage-write-removal)| Vulnerability to storage write removal | 4 |
|[[L-06]](#l-06-payable-function-does-not-transfer-eth)| `payable` function does not transfer ETH | 5 |
|[[L-07]](#l-07-nft-ownership-doesnt-support-hard-forks)| NFT ownership doesn't support hard forks | 1 |
|[[L-08]](#l-08-use-of-abiencodewithsignatureabiencodewithselector-instead-of-abiencodecall)| Use of `abi.encodeWithSignature`/`abi.encodeWithSelector` instead of `abi.encodeCall` | 2 |
|[[L-09]](#l-09-lack-of-two-step-update-for-updating-protocol-addresses)| Lack of two-step update for updating protocol addresses | 1 |

Total: 19 instances over 9 issues.

## Low Risk Issues

---

### [L-01] A user can burn their position before the nft manager transfers it

A user could call `burn_position` before the NftManager calls `transfer_position`.

This will result in the total `owned_positions` underflowing after the second call:

```rust
    fn remove_position(&mut self, owner: Address, id: U256) {
        // remove owner
        self.position_owners.setter(id).erase();

        // decrement count
        let owned_positions_count = self.owned_positions.get(owner) - U256::one();
        self.owned_positions
            .setter(owner)
            .set(owned_positions_count);
    }
```

After the manager transfer the position, the original user will have `U256::MAX` owned positions.

*There is 1 instance of this issue.*


```rust
File: pkg/seawater/src/lib.rs

    pub fn transfer_position_E_E_C7_A3_C_D(
        &mut self,
        id: U256,
        from: Address,
        to: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(msg::sender(), self.nft_manager.get(), Error::NftManagerOnly);

->      self.remove_position(from, id);
        self.grant_position(to, id);

        #[cfg(feature = "log-events")]
        evm::log(events::TransferPosition { from, to, id });

        Ok(())
    }
```
[[554-569](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L554-L569)]


---

### [L-02] A pool can be re-initialized by setting the price to zero

A pool should be able to be initialized only once with the init function:

```rust
    pub fn init(
        &mut self,
        price: U256,
        fee: u32,
        tick_spacing: u8,
        max_liquidity_per_tick: u128,
    ) -> Result<(), Revert> {
        assert_eq_or!(
            self.sqrt_price.get(),
            U256::ZERO,
            Error::PoolAlreadyInitialised
        );

        self.sqrt_price.set(price);
        self.cur_tick
            .set(I32::lib(&tick_math::get_tick_at_sqrt_ratio(price)?));

        self.fee.set(U32::lib(&fee));
        self.tick_spacing.set(U8::lib(&tick_spacing));
        self.max_liquidity_per_tick
            .set(U128::lib(&max_liquidity_per_tick));

        Ok(())
    }
```

The issue is that it's possible to re-initialize a pool by manually setting the price to zero. This shouldn't be possible; consider using an `initialized` flag instead, or remove the `set_sqrt_price` function.

*There is 1 instance of this issue.*


```rust
File: pkg/seawater/src/pool.rs

    pub fn set_sqrt_price(&mut self, new_price: U256) {
        self.sqrt_price.set(new_price);
    }
```
[[624](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L624)]


---

### [L-03] `mod` operation doesn't revert on overflow in release mode

The function `mul_mod` reverts only in debug mode when an overflow occurs, because `debug_assert` is used instead of `assert`.

*There is 1 instance of this issue.*


```rust
File: pkg/seawater/src/maths/full_math.rs

pub fn mul_mod(a: U256, b: U256, mut modulus: U256) -> U256 {
    if modulus == U256::ZERO {
        return U256::ZERO;
    }

    // alloc a 512 bit result
    let mut product = [0; 8];
    let overflow = ruint::algorithms::addmul(&mut product, a.as_limbs(), b.as_limbs());
    debug_assert!(!overflow);

    // compute modulus
    // SAFETY - ruint code
    unsafe { ruint::algorithms::div(&mut product, modulus.as_limbs_mut()) };

    modulus
}
```
[[22](https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/maths/full_math.rs#L22)]


---

### [L-04] File allows a version of solidity that is susceptible to .selector-related optimizer bug

In solidity versions prior to 0.8.21, there is a legacy code generation [bug](https://soliditylang.org/blog/2023/07/19/missing-side-effects-on-selector-access-bug/) where if `foo().selector` is called, `foo()` doesn't actually get evaluated. It is listed as low-severity, because projects usually use the contract name rather than a function call to look up the selector.

*There are 3 instances of this issue.*


```solidity
File: pkg/sol/OwnershipNFTs.sol

56: 		            SEAWATER.positionOwnerD7878480.selector,

93: 		            data != IERC721TokenReceiver.onERC721Received.selector,

173: 		            SEAWATER.positionBalance4F32C7DB.selector,
```
[[56](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L56), [93](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L93), [173](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L173)]


---

### [L-05] Vulnerability to storage write removal

This bug was introduced in Solidity version 0.8.13, and it was fixed in version 0.8.17. This bug is significantly easier to trigger with optimized via-IR code generation, but can theoretically also occur in optimized legacy code generation.. More info can be read in this [post](https://blog.soliditylang.org/2022/09/08/storage-write-removal-before-conditional-termination/).

*There are 4 instances of this issue.*


```solidity
File: pkg/sol/SeawaterAMM.sol

43: 		        assembly {

133: 		        assembly {

148: 		            case 0 {

151: 		            default {
```
[[43](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L43), [133](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L133), [148](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L148), [151](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L151)]


---

### [L-06] `payable` function does not transfer ETH

The following functions can be called by any user, who may also send some funds by mistake. In that case, those funds will be lost (this also applies to delegatecalls, in case they don't use the transferred ETH).

*There are 5 instances of this issue.*


```solidity
File: pkg/sol/OwnershipNFTs.sol

118: 		    function transferFrom(

129: 		    function transferFrom(

139: 		    function safeTransferFrom(

149: 		    function safeTransferFrom(

160: 		    function approve(address _approved, uint256 _tokenId) external payable {
```
[[118](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L118), [129](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L129), [139](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L139), [149](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L149), [160](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160)]


---

### [L-07] NFT ownership doesn't support hard forks

To ensure clarity regarding the ownership of the NFT on a specific chain, it is recommended to add `require(block.chainid == <chainIdValue>, "Invalid Chain")` in the functions below.

Alternatively, consider including the chain ID in the URI itself. By doing so, any confusion regarding the chain responsible for owning the NFT will be eliminated.

*There is 1 instance of this issue.*


```solidity
File: pkg/sol/OwnershipNFTs.sol

182: 		    function tokenURI(uint256 /* _tokenId */) external view returns (string memory) {
183: 		        return TOKEN_URI;
184: 		    }
```
[[182-184](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L182-L184)]


---

### [L-08] Use of `abi.encodeWithSignature`/`abi.encodeWithSelector` instead of `abi.encodeCall`

Consider refactoring the code by using `abi.encodeCall` instead of `abi.encodeWithSignature`/`abi.encodeWithSelector`, as the former keeps the code [typo/type safe](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3693).

*There are 2 instances of this issue.*


```solidity
File: pkg/sol/OwnershipNFTs.sol

55: 		        (bool ok, bytes memory rc) = address(SEAWATER).staticcall(abi.encodeWithSelector(

172: 		        (bool ok, bytes memory rc) = address(SEAWATER).staticcall(abi.encodeWithSelector(
```
[[55](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L55), [172](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L172)]


---

### [L-09] Lack of two-step update for updating protocol addresses

Add a two-step process for any critical address changes.

*There is 1 instance of this issue.*


```solidity
File: pkg/sol/SeawaterAMM.sol

104: 		    function updateProxyAdmin(address newAdmin) public onlyProxyAdmin {
105: 		        _setProxyAdmin(newAdmin);
106: 		    }
```
[[104-106](https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L104-L106)]


---
