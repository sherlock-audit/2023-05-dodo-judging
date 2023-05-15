roguereddwarf

medium

# MarginTrading.sol: Attacker can manipulate DODO trading pairs such that margin trades cannot be closed

## Summary
The DODO Margin Trading protocol integrates with the core DODO protocol to perform swaps.
Swaps are needed to open and close margin trades.

The intended tokens to be used are the following:
![2023-05-12_13-24](https://github.com/sherlock-audit/2023-05-dodo-roguereddwarf/assets/118631472/56e4dd9f-1fb6-4d34-948e-c14b33aa856b)

The issue is that these swap pairs are not liquid. Some of them have a TVL of 0:
https://info.dodoex.io/all
![2023-05-12_13-26](https://github.com/sherlock-audit/2023-05-dodo-roguereddwarf/assets/118631472/3d1c52ab-d3b6-4b46-8af2-dd8e68f29a01)

One might think that this is an issue due to potential slippage. This is true but it's not the main concern here. The user can specify a minimum output amount for the DODO swap.

However there is another issue related to this.

## Vulnerability Detail
An attacker might provide liquidity to an illiquid swap pair such that users can open margin trading positions:
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L257-L279

When the attacker then withdraws his liquidity the users cannot close their margin trading positions anymore since closing a trade does also require a swap:
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L294-L356

## Impact
The slippage is one concern. The bigger concern is that the attacker can manipulate swap pairs such that margin trades cannot be closed anymore and users incur losses in their trades without being able to close them.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L257-L279

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L294-L356

## Tool used
Manual Review

## Recommendation
Only allow those margin trades to be opened such that the corresponding swap pairs have sufficient liquidity. Sufficient liquidity means that the swap pair cannot be manipulated by an attacker.