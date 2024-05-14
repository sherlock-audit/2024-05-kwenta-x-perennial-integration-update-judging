Spare Pine Jay

high

# `_marketWithdraw:market.update`  is following wrong implementation of Fixed6.sol.

## Summary
`_marketWithdraw:market.update`  is following wrong implementation of [Fixed6.sol](https://github.com/equilibria-xyz/root/blob/f4b832b22da8c9969e7dc5d8e9c4163f51f45ce8/contracts/number/types/Fixed6.sol#L35)
## Vulnerability Detail
`market.update` follows [Fixed6.sol](https://github.com/equilibria-xyz/root/blob/f4b832b22da8c9969e7dc5d8e9c4163f51f45ce8/contracts/number/types/Fixed6.sol#L35) for variables and parameters.
In the current implementation of `_marketWithdraw` `withdrawal` is a `UFixed6` variable as shown below: 

```solidity
/// @notice Withdraws `withdrawal` from `account`'s `market` position
    /// @param market Market to withdraw from
    /// @param account Account to withdraw from
    /// @param withdrawal Amount to withdraw
    function _marketWithdraw(IMarket market, address account, UFixed6 withdrawal) private {
        market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.from(-1, withdrawal), false);
    }
```
In this implementation `market.update` uses `Fixed6Lib.from(-1, withdrawal)` to update the withdrawn amount.
Here lies our issue, you see in the [Fixed6.sol](https://github.com/equilibria-xyz/root/blob/f4b832b22da8c9969e7dc5d8e9c4163f51f45ce8/contracts/number/types/Fixed6.sol#L35)

```solidity
    /**
     * @notice Creates a signed fixed-decimal from a sign and an unsigned fixed-decimal
     * @param s Sign
     * @param m Unsigned fixed-decimal magnitude
     * @return New signed fixed-decimal
     */
    function from(int256 s, UFixed6 m) internal pure returns (Fixed6) {
        if (s > 0) return from(m);
        if (s < 0) {
            // Since from(m) multiplies m by BASE, from(m) cannot be type(int256).min
            // which is the only value that would overflow when negated. Therefore,
            // we can safely negate from(m) without checking for overflow.
            unchecked { return Fixed6.wrap(-1 * Fixed6.unwrap(from(m))); }
        }
        return ZERO;
    }
  ```
Here, according to the natspec, it assumes that the `from(m)` will be multiplied by *Base* making it immune to *overflow*. The function it talks about is [this](https://github.com/equilibria-xyz/root/blob/f4b832b22da8c9969e7dc5d8e9c4163f51f45ce8/contracts/number/types/Fixed6.sol#L62):
  
```solidity 
    /**
     * @notice Creates a signed fixed-decimal from a signed integer
     * @param a Signed number
     * @return New signed fixed-decimal
     */
    function from(int256 a) internal pure returns (Fixed6) {
        return Fixed6.wrap(a * BASE);
    }
 ```
 But the current implementation of `market.update` follows this [function](https://github.com/equilibria-xyz/root/blob/f4b832b22da8c9969e7dc5d8e9c4163f51f45ce8/contracts/number/types/Fixed6.sol#L34) due to the input `withdrawal` parameter being a `UFixed6` variable hence making it prone to overflow.
 
 ```solidity
     /**
     * @notice Creates a signed fixed-decimal from an unsigned fixed-decimal
     * @param a Unsigned fixed-decimal
     * @return New signed fixed-decimal
     */
    function from(UFixed6 a) internal pure returns (Fixed6) {
        uint256 value = UFixed6.unwrap(a);
        if (value > uint256(type(int256).max)) revert Fixed6OverflowError(value);
        return Fixed6.wrap(int256(value));
    }
 ```

## Impact
Incorrect updates and accounting issues in `_chargeFee`, `_raiseKeeperFee`, `_marketSettle`.
## Code Snippet
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L455
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L462
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L414
## Tool used

Manual Review

## Recommendation
- Make a custome type or  introduce checks for overflow issues.