# Issue H-1: MarginTrading.sol: Missing flash loan initiator check allows attacker to open trades, close trades and steal funds 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/34 

## Found by 
0xHati, BAHOZ, Bauer, BowTiedOriole, CRYP70, Jiamin, Juntao, Quantish, Tendency, VAD37, alexzoid, carrotsmuggler, circlelooper, curiousapple, nobody2018, oot2k, pengun, qbs, roguereddwarf, rvierdiiev, sashik\_eth, shaka, shogoki, smiling\_heretic, theOwl
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



## Discussion

**Zack995**

I agree with the severity and error of this issue

+        require(_initiator == address(this));
         
We will adopt this recommendation to address the issue.

# Issue H-2: Missing checks in 2 places, allows proxy holders to drain the margin account 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/104 

## Found by 
0x2e, BAHOZ, J4de, ctf\_sec, curiousapple, jprod15, n33k, pengun, shogoki, simon135
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



## Discussion

**Zack995**

These areas do indeed have the potential for unlimited approval and swap risks. However, currently, the front-end entry point only supports authorization for our robot to execute. The purpose is to authorize the execution of opening and closing positions automatically. Regardless of whether the permissions for unlimited approval and swap are restricted, it is still possible to perform many bold actions, such as depleting assets, large flash loans, and various other behaviors. Therefore, authorization is considered a trusted action.

**roguereddwarf**

The role of the proxy has been clearly described in the README
```
There is a proxy role that has permissions stored in the ALLOWED_FLASH_LOAN structure. This role can execute opening or closing positions on behalf of the user to achieve stop loss or take profit objectives.
```

A user that sets a proxy will give the proxy more privileges than intended.
The sponsor rightfully pointed out that this issue does not significantly increase the privileges of the proxy since opening and closing trades is inherently a sensitive operation and the proxy is trusted by the user.
Still this increases the privileges of the proxy and can hence be considered a valid issue.

# Issue M-1: MarginTradingFactory.sol: User can steal ETH that is stuck in contract that should be rescued by the owner 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/24 

## Found by 
0xHati, BAHOZ, BowTiedOriole, evilakela, jprod15, nobody2018, roguereddwarf, rvierdiiev, simon135, smiling\_heretic
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



## Discussion

**Zack995**

The user mistakenly deposited this amount of money into MarginTradingFactory, and the owner did not incur any losses. Therefore, the severity of the issue is not high. To prevent this operation in the future, the receive() method will be removed.

# Issue M-2: MarginTrading.sol: The whole balance and not just the traded funds are deposited into Aave when a trade is opened 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/72 

## Found by 
roguereddwarf
## Summary
It's expected by the protocol that funds can be in the `MarginTrading` contract without being deposited into Aave as margin.

We can see this by looking at the `MarginTradingFactory.depositMarginTradingETH` and `MarginTradingFactory.depositMarginTradingERC20` functions.

If the user sets `margin=false` as the parameter, the funds are only sent to the `MarginTrading` contract but NOT deposited into Aave.

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L259-L272

So clearly there is the expectation for funds to be in the `MarginTrading` contract that should not be deposited into Aave.

This becomes an issue when a trade is opened.

## Vulnerability Detail
Let's look at the `MarginTrading._openTrade` function that is called when a trade is opened:

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L257-L279

The whole balance of the token will be deposited into Aave:

```solidity
_tradeAmounts[i] = IERC20(_tradeAssets[i]).balanceOf(address(this)); 
_lendingPoolDeposit(_tradeAssets[i], _tradeAmounts[i], 1); 
```

Not just those funds that have been acquired by the swap. This means that funds that should stay in the `MarginTrading` contract might also be deposited as margin.

## Impact
When opening a trade funds can be deposited into Aave unintentionally. Thereby the funds act as margin and the trade can incur a larger loss than expected.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L259-L272

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L257-L279

## Tool used
Manual Review

## Recommendation
It is necessary to differentiate the funds that are acquired by the swap and those funds that were there before and should stay in the contract:

```diff
diff --git a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
index f68c1f3..42f96cf 100644
--- a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
+++ b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
@@ -261,6 +261,10 @@ contract MarginTrading is OwnableUpgradeable, IMarginTrading, IFlashLoanReceiver
         bytes memory _swapParams,
         address[] memory _tradeAssets
     ) internal {
+        int256[] memory _amountsBefore = new uint256[](_tradeAssets.length);
+        for (uint256 i = 0; i < _tradeAssets.length; i++) {
+            _amountsBefore[i] = IERC20(_tradeAssets[i]).balanceOf(address(this));
+        }
         if (_swapParams.length > 0) {
             // approve to swap route
             for (uint256 i = 0; i < _swapApproveToken.length; i++) {
@@ -272,8 +276,10 @@ contract MarginTrading is OwnableUpgradeable, IMarginTrading, IFlashLoanReceiver
         }
         uint256[] memory _tradeAmounts = new uint256[](_tradeAssets.length);
         for (uint256 i = 0; i < _tradeAssets.length; i++) {
-            _tradeAmounts[i] = IERC20(_tradeAssets[i]).balanceOf(address(this));
-            _lendingPoolDeposit(_tradeAssets[i], _tradeAmounts[i], 1);
+            if (_amountsBefore[i] < IERC20(_tradeAssets[i]).balanceOf(address(this))) {
+                _tradeAmounts[i] = IERC20(_tradeAssets[i]).balanceOf(address(this)) - _amountsBefore[i];
+                _lendingPoolDeposit(_tradeAssets[i], _tradeAmounts[i], 1);
+            }
         }
         emit OpenPosition(_swapAddress, _swapApproveToken, _tradeAssets, _tradeAmounts);
     }
```

If funds that were in the contract prior to the swap should be deposited there is the separate `MarginTrading.lendingPoolDeposit` function to achieve this.



## Discussion

**Zack995**

In terms of product design, users do not have a separate concept of balance. However, the contract is designed to be more flexible and allows for balances to be maintained. Users will not perceive or interact with balances in terms of user experience or operations.

**roguereddwarf**

Based on the smart contract logic there is clearly the notion of balance that is not intended to be used as collateral (but e.g. used to repay a loan).
If this notion of a separate balance is not exposed on the front-end this is not a sufficient mitigation of the issue since the issue is clearly present in the smart contract.

# Issue M-3: MarginTrading.sol: When partially closing a trade all tokens are used to pay back debt and not only the swapped tokens 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/80 

## Found by 
roguereddwarf
## Summary
It's expected by the protocol that funds can be in the `MarginTrading` contract without being used for anything.

We can see this by looking at the `MarginTradingFactory.depositMarginTradingETH` and `MarginTradingFactory.depositMarginTradingERC20` functions.

If the user sets `margin=false` as the parameter, the funds are only sent to the `MarginTrading` contract but NOT deposited into Aave.

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L259-L272

The purpose of this is that the user can then call `lendingPoolDeposit` or `lendingPoolRepay` to add margin or close a trade.

When a trade is partially closed by taking out a flash loan and swapping it for the debt token all the funds are used to pay back debt but it should be restricted to the funds that were swapped.

## Vulnerability Detail
When partially closing a trade (`flag=0`) the `MarginTrading._closetrade` function is called and `_tradeAmounts` is set like this:

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L320-L323

So it's set to the full balance, and this amount is then used to repay the debt:

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L325-L327

So this does not leave those funds idle that were in the contract before.

## Impact
When partially closing a trade the amount that is repaid can be bigger than expected. So the funds that should remain in the contract for other purposes are used up and cannot be used otherwise.

Also since more debt is repaid than intended the trade is altered in an unintended way and will now be liquidated at a different price which is not what the user wants.

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L203-L211

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L259-L272

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L294-L356

## Tool used
Manual Review

## Recommendation
Only use swapped funds to partially repay debt.

```diff
diff --git a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
index f68c1f3..29176f9 100644
--- a/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
+++ b/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol
@@ -318,8 +318,14 @@ contract MarginTrading is OwnableUpgradeable, IMarginTrading, IFlashLoanReceiver
                 _tradeAmounts[i] = (IERC20(_debtTokens[i]).balanceOf(address(this)));
             }
         } else {
+            int256[] memory _amountsBefore = new uint256[](_tradeAssets.length);
             for (uint256 i = 0; i < _tradeAssets.length; i++) {
-                _tradeAmounts[i] = (IERC20(_tradeAssets[i]).balanceOf(address(this)));
+                _amountsBefore[i] = IERC20(_tradeAssets[i]).balanceOf(address(this));
+            }
+            for (uint256 i = 0; i < _tradeAssets.length; i++) {
+                if (_tradeAmounts[i] > IERC20(_tradeAssets[i]).balanceOf(address(this))) {
+                    _tradeAmounts[i] = (IERC20(_tradeAssets[i]).balanceOf(address(this))) - _amountsBefore[i];
+                }
             }
         }
         for (uint256 i = 0; i < _tradeAssets.length; i++) {
```



## Discussion

**Zack995**

In terms of product design, users do not have a separate concept of balance. However, the contract is designed to be more flexible and allows for balances to be maintained. Users will not perceive or interact with balances in terms of user experience or operations.

**roguereddwarf**

Based on the smart contract logic there is clearly the notion of balance that is not intended to be used as collateral (but e.g. used to repay a loan).
If this notion of a separate balance is not exposed on the front-end this is not a sufficient mitigation of the issue since the issue is clearly present in the smart contract.

# Issue M-4: `createMarginTrading` of MarginTradingFactory wont work on actual mainnet 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/103 

## Found by 
curiousapple, roguereddwarf
## Summary
As stated here 
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L107
Dodo defines `createMarginTrading()` to allow users to create a margin trading account, deposit funds, and open a position.
However incorrect implementation of it won't allow the user to do so, **once it's deployed on the actual mainnet.** 

## Vulnerability Detail
There are three steps in `createMarginTrading()`.
**Step 1: Clone** :heavy_check_mark: 
**Step 2:  Deposit** :warning: 
Here if we look closely, dodo is always passing margin boolean as false for both types of deposits, hence these tokens are only transferred to the actual account, and `lendingPoolDeposit` is never done.
Hence there is no collateral in the name of the margin account in the lending pool yet.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L144-L148

**Step 3: Open Trade** :negative_squared_cross_mark: 
The user passes rate mode as 1 or 2 in hopes of opening a position and incurring a debt.
However, since there is no collateral to borrow against, it would revert to Aave's side.
Stopping users from creating margin accounts.

The complete trace on aave's side is as per follows 
1. Flashloan (mode = 1,2)
https://github.com/aave/protocol-v2/blob/ce53c4a8c8620125063168620eba0a8a92854eb8/contracts/protocol/lendingpool/LendingPool.sol#L483
2. _executeBorrow
https://github.com/aave/protocol-v2/blob/ce53c4a8c8620125063168620eba0a8a92854eb8/contracts/protocol/lendingpool/LendingPool.sol#L540-L542
3. ValidationLogic.validateBorrow
https://github.com/aave/protocol-v2/blob/ce53c4a8c8620125063168620eba0a8a92854eb8/contracts/protocol/lendingpool/LendingPool.sol#L866
4. REVERT
https://github.com/aave/protocol-v2/blob/ce53c4a8c8620125063168620eba0a8a92854eb8/contracts/protocol/libraries/logic/ValidationLogic.sol#L168
`require(vars.userCollateralBalanceETH > 0, Errors.VL_COLLATERAL_BALANCE_IS_0);`

Whats interesting about this issue is, DODO do have a test to test this functionality, and expect it to work.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/test/marginTrading/MarginTradingFactory.t.sol#L35
However this issue is missed in testing, since they are using mocked lending pool without verifcations on collateral.

## Impact
1. Breaks project assumptions.
Protocol mentions it clearly that they would like users to create, deposit, and open trade using the concerned function, but due to this issue, users can not. 
They have even written a test to confirm it but missed the validations in mocked setup.

2. Users being unaware would waste a significant amount of gas in failed calls to createMarginTrading()

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L115

## Tool used

Manual Review

## Recommendation
Consider passing margin boolean as true, so there is collateral to borrow against.
Also, consider adding forking mainnet to your test suite



## Discussion

**Zack995**

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L273
The margin is stored on the balance of the marginTrading contract and will be deposited together with the swap into Aave to serve as collateral, achieving a successful position opening.

**roguereddwarf**

Based on the smart contract there is clearly the intention of opening a position when `createMarginTrading` is called. Since the flag is set to "false" the position is not opened which does not expose the user to price action as he intends to.
This can therefore be considered a valid Medium in spite of the sponsor's comment.

# Issue M-5: MarginTrading.sol: Attacker can manipulate DODO trading pairs such that margin trades cannot be closed 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/138 

## Found by 
roguereddwarf, theOwl
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



## Discussion

**Zack995**

Aave supports only mainstream tokens, and the frontend trading pairs for opening positions are controlled by the backend. Additionally, our swap router has price protection, which prevents opening positions for trading pairs with insufficient liquidity. Therefore, it is not possible to open positions with long-tail tokens. If users manage to bypass the DODO platform and attempt to open positions with such tokens, it would be their own responsibility and action.

**ctf-sec**

the link is broken in the original report, just want to help add the image attached by the senior watson

![image](https://github.com/sherlock-audit/2023-05-dodo-judging/assets/114844362/48175216-aa8b-4db7-955f-33757dbf5662)


**roguereddwarf**

This issue is considered valid in spite of the sponsors comment since front-end restrictions are not considered a sufficient mitigation for the issue in the smart contract.

In the contest README the supported tokens were listed (including among others WMATIC and WETH).
Clearly some of the Trading Pairs allow for the described attack because of insufficient liquidity and a user interacting with the smart contracts with the intended tokens can encounter this issue.

# Issue M-6: `MarginTrading.sol` can't be upgradable. 

Source: https://github.com/sherlock-audit/2023-05-dodo-judging/issues/152 

## Found by 
GimelSec
## Summary

`MarginTradingFactory` uses `Clones.cloneDeterministic` to create a new `MarginTrading` contract. However, `MarginTrading` seems to be an upgradeable contract.  But `Clone` is not compatible with upgradeable contracts. 

https://docs.openzeppelin.com/contracts/3.x/api/proxy
> The Clones library provides a way to deploy minimal non-upgradeable proxies for cheap. This can be useful for applications that require deploying many instances of the same contract (for example one per user, or one per task). These instances are designed to be both cheap to deploy, and cheap to call. The drawback being that they are not upgradeable.

## Vulnerability Detail

`MarginTradingFactory` uses `Clones.cloneDeterministic` to create a new `MarginTrading`.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L121
```solidity
    function createMarginTrading(
        uint8 _flag,
        bytes calldata depositParams,
        bytes calldata executeParams
    ) external payable returns (address marginTrading) {
        if (_flag == 1) {
            marginTrading = Clones.cloneDeterministic(
                MARGIN_TRADING_TEMPLATE,
                keccak256(abi.encodePacked(msg.sender, crossMarginTrading[msg.sender].length, _flag))
            );
            …
    }
```

And `MarginTrading` seems to be an upgradeable contract.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L19
```solidity
contract MarginTrading is OwnableUpgradeable, IMarginTrading, IFlashLoanReceiver {
```


The answer in this post explains why `Clone` is not compatible with upgradeable contracts. https://forum.openzeppelin.com/t/contract-factory-for-upgradeable-erc721/11153
> The problem is that the implementation address for the transparent proxy is kept in the storage of the transparent proxy, but when the user goes through the clone it will be using the storage of the clone, so it doesn’t know where to delegate.



## Impact

If MARGIN_TRADING_TEMPLATE is an upgradeable contract, `Clones.cloneDeterministic` only returns the address of the transparent proxy without the actual implementation.

## Code Snippet

https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L121
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L129
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L19


## Tool used

Manual Review

## Recommendation

Don’t use an upgradeable contract as the implement address in `Clone`. If DODO want a upgradeability mechanism for `MarginTrading`, consider using [Beacon](https://docs.openzeppelin.com/contracts/3.x/api/proxy#beacon)



## Discussion

**Zack995**

The purpose of adding the OwnableUpgradeable contract was to facilitate the initialization of the owner through the init method in the future. However, this has caused some confusion. We are now planning to replace it with our own custom InitializableOwnable contract. Here is the code:
```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.15;

/**
 * @title Ownable
 * @author DODO Breeder
 * @notice Ownership related functions
 */
contract InitializableOwnable {
    address public _OWNER_;
    address public _NEW_OWNER_;
    bool internal _INITIALIZED_;

    // ============ Events ============

    event OwnershipTransferPrepared(address indexed previousOwner, address indexed newOwner);

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    // ============ Modifiers ============

    modifier notInitialized() {
        require(!_INITIALIZED_, "DODO_INITIALIZED");
        _;
    }

    modifier onlyOwner() {
        require(msg.sender == _OWNER_, "NOT_OWNER");
        _;
    }

    // ============ Functions ============

    function initOwner(address newOwner) public notInitialized {
        _INITIALIZED_ = true;
        _OWNER_ = newOwner;
    }

    function transferOwnership(address newOwner) public onlyOwner {
        emit OwnershipTransferPrepared(_OWNER_, newOwner);
        _NEW_OWNER_ = newOwner;
    }

    function claimOwnership() public {
        require(msg.sender == _NEW_OWNER_, "INVALID_CLAIM");
        emit OwnershipTransferred(_OWNER_, _NEW_OWNER_);
        _OWNER_ = _NEW_OWNER_;
        _NEW_OWNER_ = address(0);
    }
}
```

**roguereddwarf**

The sponsor is correct that when there is no Proxy used and the initializable contract is only used in order to have the functionality to initialize the owner, there is no issue.

However the Watson pointed to a valid issue under the assumption that a Proxy will be used. Since the additional information provided by the sponsor was not available at the time of the contest, this issue can be considered valid.

