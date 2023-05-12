curiousapple

high

# Missing checks in 2 places, allows proxy holders to drain the margin account

## Summary
Missing checks in 2 places, allows proxy holders to drain the margin account.

## Vulnerability Detail
Dodo defines the proxy role with the following intent 

> Q: Are there any additional protocol roles? If yes, please explain in detail:
> There is a proxy role that has permissions stored in the ALLOWED_FLASH_LOAN structure. This role can execute opening or closing positions on behalf of the user to achieve stop loss or take profit objectives.

**The intent here is that users allow proxy addresses to act on their behalf, to their benefit.**
There are checks in multiple places to enforce above, that beneficiary remains margin account only even if action is being executed by proxy.

For example
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L376
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L365

However, there are two places where checks are missed, exposing funds at proxy's mercy :) 

1. `_swapApproveTarget`
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L308-L310
The proxy holder can pass their own address as `_swapApproveTarget` and get an infinite allowance on user funds.

> Attack: 
> flag: 1 
> _swapApproveToken[]: all tokens margin account holds
> _swapAddress: some dummy contract with fallback
> tradeAssets: empty
> withdrawAssets: empty

2. `_swapAddress`
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L312
This line basically gives complete ad-hoc access to the proxy, using a combination of `_swapAddress` and `_swapParams`, they can basically do anything, eg: transfer tokens, and approve them.....

_Note: This is applicable for both open and close trade_

## Impact
Loss of user funds
The trust assumption of DODO considering proxy access is broken.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L308
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L312

## Tool used
Manual Review

## Recommendation
Consider validating swap parameters 