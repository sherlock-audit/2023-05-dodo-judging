roguereddwarf

high

# MarginTrading.sol: Missing flash loan initiator check allows attacker to open trades, close trades and steal funds

## Summary
The `MarginTrading.executeOperation` function is called when a flash loan is made (and it can only be called by the `lendingPool`).

The wrong assumption by the protocol is that the flash loan can only be initiated by the `MarginTrading` contract itself.

However this is not true. A flash loan can be initiated for any `receiverAddress`.

This is actually a known mistake that devs make and the aave docs warn about this (although admittedly the warning is not very clear):
https://docs.aave.com/developers/v/2.0/guides/flash-loans

![2023-05-11_12-43](https://github.com/sherlock-audit/2023-05-dodo-roguereddwarf/assets/118631472/1bc59eb4-407b-4b5f-a38b-9c415932caf1)

So an attacker can execute a flash loan with the `MarginTrading` contract as `receiverAddress`. Also the funds that are needed to pay back the flash loan are pulled from the `receiverAddress` and NOT from the `initiator`:

https://github.com/aave/protocol-v2/blob/30a2a19f6d28b6fb8d26fc07568ca0f2918f4070/contracts/protocol/lendingpool/LendingPool.sol#L532-L536

This means the attacker can close a position or repay a position in the `MarginTrading` contract.

By crafting a malicious swap, the attacker can even steal funds.

## Vulnerability Detail
Let's assume there is an ongoing trade in a `MarginTrading` contract:

```text
daiAToken balance = 30000
wethDebtToken balance = 10

The price of WETH when the trade was opened was ~ 3000 DAI
```

In order to profit from this the attacker does the following (not considering fees for simplicity):
1. Take a flash loan of 30000 DAI with `MarginTrading` as `receiverAddress` with `mode=0` (flash loan is paid back in the same transaction)
2. Price of WETH has dropped to 2000 DAI. The attacker uses a malicious swap contract that pockets 10000 DAI for the attacker and swaps the remaining 20000 DAI to 10 WETH (the attacker can freely choose the swap contract in the `_params` of the flash loan).
3. The 10 WETH debt is repaid
4. Withdraw 30000 DAI from Aave to pay back the flash loan



## Impact
The attacker can close trades, partially close trades and even steal funds.

(Note: It's not possible for the attacker to open trades because he cannot incur debt on behalf of the `MarginTrading` contract)

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L121-L166

https://github.com/aave/protocol-v2/blob/30a2a19f6d28b6fb8d26fc07568ca0f2918f4070/contracts/protocol/lendingpool/LendingPool.sol#L481-L562

## Tool used
Manual Review

## Recommendation
The fix is straightforward:

```diff
diff --git a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
index f68c1f3..5b4b485 100644
--- a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
+++ b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
@@ -125,6 +125,7 @@ contract MarginTrading is OwnableUpgradeable, IMarginTrading, IFlashLoanReceiver
         address _initiator,
         bytes calldata _params
     ) external override onlyLendingPool returns (bool) {
+        require(_initiator == address(this));
         //decode params exe swap and deposit
         {
```

This ensures that the flash loan has been initiated by the `MarginTrading.executeFlashLoans` function which is the intended initiator.