Fantastic Misty Ram

medium

# _commitPrice doesn't work on user's behalf as expected if the msg.sender is an operator.

## Summary
The operator role is limited to only being able to send multiinvoker transactions for the user on user's behalf via the `invoke(account, actions)` function
However, for the COMMIT_PRICE action case, the operator is not sending the transaction on user's behalf.

## Vulnerability Detail
The operator can invoke the COMMIT_PRICE action on behalf of the user.
```solidity
    function invoke(address account, Invocation[] calldata invocations) external payable {
        _invoke(account, invocations);
    }
```
In this case `msg.sender` is an operator and `account` is the user.
`_commitPrice` function doesn't have `account` parameter for transferring the keeper fee.
And it doesn't have the code for receiving the ETH value from the user.

https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L346~L362
```solidity
    function _commitPrice(
        address oracleProviderFactory,
        uint256 value,
        bytes32[] memory ids,
        uint256 version,
        bytes memory data,
        bool revertOnFailure
    ) internal {
        UFixed18 balanceBefore = DSU.balanceOf();

        try IPythFactory(oracleProviderFactory).commit{value: value}(ids, version, data) { // @audit if msg.sender is an operator, `value` ETH didn't come from the user.
            // Return through keeper fee if any
            DSU.push(msg.sender, DSU.balanceOf().sub(balanceBefore)); // @audit if msg.sender is an operator, it's not transferring DSU to the user.
        } catch (bytes memory reason) {
            if (revertOnFailure) Address.verifyCallResult(false, reason, "");
        }
    }
```
## Impact
When the operator invokes the `_commitPrice` function on the user's behalf, the `value` ETH should come from the user, and the DSU fee should also be transferred to the user.

## Code Snippet

_commitPrice function
https://github.com/sherlock-audit/2024-05-kwenta-x-perennial-integration-update/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L346~L362

## Tool used
Manual Review

## Recommendation
It's recommended to disable the `_commitPrice` function if it's an operator or implement the code to transfer the DSU fee to the user and receive the ETH from the user.
