curiousapple

medium

# DODO margin accounts can not participate in aave's integration referral program if it is restarted again

## Summary
 DODO margin accounts can not participate in aave's integration referral program if it is restarted again

## Vulnerability Detail
Aave had a referral program to incentivize its integrations.
To be eligible you had to pass a referral code in smart contract actions done on aave.
It's inactive as of now, but **it could be restarted again** by AAVE governance  
https://docs.aave.com/developers/v/2.0/the-core-protocol/lendingpool#deposit
![image](https://github.com/sherlock-audit/2023-05-dodo-abhishekvispute/assets/46760063/1aa43b9c-c4e6-4bce-b6a4-53350fe659a4)

However in the case of DODO, since the referral code is hardcoded to 0, even if the program is restarted, dodo margin accounts won't be able to participate.
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/Types.sol#L10
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/MarginTrading.sol#L376

## Impact
Loss of revenue generated through aave referral program

## Code Snippet
https://github.com/sherlock-audit/2023-05-dodo/blob/main/dodo-margin-trading-contracts/contracts/marginTrading/Types.sol#L10

## Tool used
Manual Review

## Recommendation
Consider making `REFERRAL_CODE`, a mutable parameter with access to the owner (factory).