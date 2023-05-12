ctf_sec

high

# Lack of token whitelisting

## Summary

Lack of whitelisting token

## Vulnerability Detail

> Q: Which ERC20 tokens do you expect will interact with the smart contracts?
> WETH USDC WBTC DAI MATIC

but there is no token whitelisting

https://app.aave.com/ if we see the AAVE interface

there are other token such as BAL, cbETH, CRV, LDO, LINK, etc...

user can set the flag to cross (1) and open trade as long as it is supported in AAVE deposit and then use it as collateral to trade.

## Impact

Lack of whitelisting token impact trading positions

## Code Snippet

https://github.com/sherlock-audit/2023-05-dodo/blob/930565dd875dac24609441423c7c76c2ae4719a8/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L208

## Tool used

Manual Review

## Recommendation

We recommend the protocol add token whitelisting