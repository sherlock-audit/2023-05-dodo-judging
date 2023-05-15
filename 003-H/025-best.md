rvierdiiev

medium

# MarginTrading doesn't have ability to withdraw eth

## Summary
MarginTrading doesn't have ability to withdraw eth
## Vulnerability Detail
MarginTrading contract has [`receive` function](https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L78) which can be used by anyone to top up contract. I think that this function should be used by WETH contract only. But now it's not restricted. So user can top up contract by himself

Also there is `withdrawETH` function.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L221C14-L228
```solidity
    function withdrawETH(bool _margin, uint256 _marginAmount, uint8 _flag) external payable onlyUser {
        if (_margin) {
            _lendingPoolWithdraw(address(WETH), _marginAmount, _flag);
        }
        WETH.withdraw(_marginAmount);
        _safeTransferETH(msg.sender, _marginAmount);
        emit WithdrawETH(_marginAmount, _margin, _flag);
    }
```

The problem here is that this function always tries to withdraw amount from WETH contract and then send them to user.
But in case if `MarginTrading` has no balance on WETH, then this call will fail. In other words it's not possible for user to receive his eth in this way.

Another thing that i would like to add to this report is that `withdrawETH` function is payable itself(for no reasons). So user can fund contract in this way as well.
## Impact
User can't withdraw eth.
## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L78
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L221C14-L228
## Tool used

Manual Review

## Recommendation
I guess that `receive` function should be callable by WETH only. Otherwise you need to create ability for user to withdraw contract's balance. Also `withdrawETH` function should not be payable.