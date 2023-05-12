roguereddwarf

high

# MarginTradingFactory.sol: User can steal ETH that is stuck in contract that should be rescued by the owner

## Summary
The `MarginTradingFactory` contract handles ETH when users deposit it into their `MarginTrading` contracts.
Due to the fact that wrong user input can cause ETH to be stuck in the contract, there is a `cleanETH` function that allows the owner to rescue the stuck ETH and refund it to the user that lost it:

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L225-L229

The issue is that an attacker can abuse the `multicall` function to steal this ETH.

## Vulnerability Detail
Assume there is 1 ETH in the `MarginTradingFactory` contract.

The attacker can do the following:
1. Call `MarginTradingFactory.multicall` with two calls to `depositMarginTradingETH` and send along 1 ETH.

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L74-L87

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

2. Both calls to `depositMarginTradingETH` are performed with `msg.value` and since there are now 1 ETH + 1 ETH = 2 ETH in the contract, both calls succeed, leaving 0 ETH in the contract. Thereby the attacker has stolen the 1 ETH which only the owner should be able to rescue

## Impact
Note: This issue is not about the fact that a wrong user input can lead to a loss of their funds (which is not a valid issue).

Instead, due to this issue an attacker can steal the ETH from the `MarginTradingFactory` contract that should only be accessible to the owner of the contract.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L225-L229

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L74-L87

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

## Tool used
Manual Review

## Recommendation
Removing the `multicall` function is the easiest remedation.
Alternatively perform additional accounting to ensure that only `msg.value` is spent.
Or check that there is only one call to `depositMarginTradingETH` in the `multicall` but keep in mind that there might be nested multicalls.