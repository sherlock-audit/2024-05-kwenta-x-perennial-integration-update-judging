Acidic Vermilion Pike

medium

# ETH remaining in the contract after `invoke` is sent to `account` rather than `msg.sender`

## Summary

Contest README specifies:

> USDC or DSU is always pulled from the account (not msg.sender). The msg.sender is expected to provide the ETH update fee required for COMMIT actions

While not explicitly stated, it's obvious that if `msg.sender` overpays ETH for whatever reason, it should be returned back to `msg.sender`. However, it's currently sent to `account`:

```solidity
        // ETH must not remain in this contract at rest
        Address.sendValue(payable(account), address(this).balance);
```

## Vulnerability Detail

The user who executes `invoke` transaction is expected to pre-calculate required amount in ETH to send to commit oracle prices, so any ETH remainder remaining in `MultiInvoker` as an error in calculations is user's mistake. However, there are some edge cases when the amount in ETH consumed can be less than what the user has sent for reasons outside user's control, such as:
- if `revertOnFailure == true` and commit reverts, then ETH will not be fully used
- chainlink `verifyBulk`'s cost in ETH depends on internal cost amount and user's personal discount, both of them can be changed by owner. If any of this changes between user submitting transaction and the transaction execution, then the actual cost might be smaller and excess of ETH will be returned back.

There might be the other reasons outside of user's control for legal ETH remainder which should be sent to keeper (`msg.sender`). However currently it's sent to `account`, so the keeper user loses this amount.

## Impact

User loses ETH overpaid for commit actions in some rare edge cases.

## Code Snippet

`_invoke` sends back any ETH remainder to `account` rather than `msg.sender`:
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L190

## Tool used

Manual Review

## Recommendation

Send ETH remainder to `msg.sender`.