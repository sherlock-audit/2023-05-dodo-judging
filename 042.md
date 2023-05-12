roguereddwarf

high

# MarginTrading.sol: liquidity rewards from Aave cannot be claimed

## Summary
The `MarginTrading` contract which manages the Aave positions of the user does not have an ability to claim liquidity rewards.

## Vulnerability Detail
Aave does currently not offer liquidity rewards but it's not safe to assume that this will be so in the future.
The Aave protocol clearly is designed to offer liquidity rewards and this can be enabled by governance.

For example it might be enabled when Aave wants to attract more liquidity providers when there's a more attractive competitor.

## Impact
Users of the DODO Margin Trading protocol will not be able to claim their Aave liquidity rewards

## Code Snippet
(Need to add code to pass the check, but there is no specific code for this issue)
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L19

## Tool used
Manual Review

## Recommendation
Here's the documentation for how to implement the claiming of liquidity rewards:
https://docs.aave.com/developers/v/2.0/guides/liquidity-mining