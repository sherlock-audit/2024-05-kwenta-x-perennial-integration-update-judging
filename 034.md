Spare Pine Jay

medium

# `market.update` could lead to integration issues in future

## Summary
`market.update` could lead to integration issues in future
## Vulnerability Detail
When Market.update is called, any parameters except protected = true will perform the following check from the InvariantLib.validate:
```solidity
if (
    !PositionLib.margined(
        context.latestPosition.local.magnitude().add(context.pending.local.pos()),
        context.latestOracleVersion,
        context.riskParameter,
        context.local.collateral
    )
) revert IMarket.MarketInsufficientMarginError();
```
This means that even updates which do not change anything (empty order and 0 collateral change) still perform this check and revert if the user's collateral is below margin requirement.
Such method to settle accounts is used in `MultiInvoker._marketWithdraw`:
```solidity
    /// @notice Withdraws `withdrawal` from `account`'s `market` position
    /// @param market Market to withdraw from
    /// @param account Account to withdraw from
    /// @param withdrawal Amount to withdraw
    function _marketWithdraw(IMarket market, address account, UFixed6 withdrawal) private {
        market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.from(-1, withdrawal), false);
    }
```
This could lead to settling issues like explained in this audit [report](https://github.com/sherlock-audit/2024-02-perennial-v2-3-judging/issues/23)

## Impact
Future integration issues 
## Code Snippet
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L455

## Tool used

Manual Review

## Recommendation
Use `market.settle`.