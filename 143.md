ctf_sec

medium

# admin owner of the protocol is label as restricted but still has too much power to transfer user's fund out!

## Summary

admin owner of the protocol is label as restricted but still has too much power

## Vulnerability Detail

Per on-chain context:

> Q: Is the admin/owner of the protocol/contracts TRUSTED or > > RESTRICTED?

While in fact the protocol owner / admin has too much power

```solidity
    function _depositMarginTradingERC20(
        address _marginTradingAddress,
        address _marginAddress,
        uint256 _marginAmount,
        bool _margin,
        uint8 _flag
    ) internal {
        require(IMarginTrading(_marginTradingAddress).user() == msg.sender, "factory:caller is not the user");
        DODOApprove.claimTokens(_marginAddress, msg.sender, _marginTradingAddress, _marginAmount);
        if (_margin) {
            IMarginTrading(_marginTradingAddress).lendingPoolDeposit(_marginAddress, _marginAmount, _flag);
        }
        emit DepositMarginTradingERC20(_marginTradingAddress, _marginAddress, _marginAmount, _margin, _flag);
    }
```

note:

```solidity
  require(IMarginTrading(_marginTradingAddress).user() == msg.sender, "factory:caller is not the user");
        DODOApprove.claimTokens(_marginAddress, msg.sender, _marginTradingAddress, _marginAmount);
```

when depositMarginTradingERC20, the DODOApprove is used to pull the fund from msg.sender

but the admin (protocol) of the DODOApprove has access to the user's fund out of side of the transaction

the admin can just upgrade proxy contract and call claimTokens to steal user's fund on behalf of user....

```solidity
    function setDODOProxy() external onlyOwner notLocked {
        emit SetDODOProxy(_DODO_PROXY_, _PENDING_DODO_PROXY_);
        _DODO_PROXY_ = _PENDING_DODO_PROXY_;
        lockSetProxy();
    }

    function claimTokens(address token, address who, address dest, uint256 amount) external {
        require(msg.sender == _DODO_PROXY_, "DODOApprove:Access restricted");
        if (amount > 0) {
            IERC20(token).safeTransferFrom(who, dest, amount);
        }
    }

```

https://docs.dodoex.io/english/developers/contract-framework#token-authorization-dodoapprove

> DODO V2 abstracts a unique user authorization contract (DODOApprove) from the contract structure. For different kinds of tokens, users only need to authorize once, and then they can smoothly execute all the platform-wide operations such as trading and liquidity management of the authorized tokens. 

> DODOApprove, as the business interaction portal of the platform, helps users to manage the security of token authorization. DODOApprove also has an added time lock mechanism. When the DODO Team upgrades or add a new proxy contract, the time lock mechanism will ensure that the operation is cooled down for 3 days, leaving enough time to publicize the contract adjustment to the community to enhance the trust in DODOApprove. 

## Impact

admin owner of the protocol is not restricted at all

## Code Snippet

https://github.com/sherlock-audit/2023-05-dodo/blob/930565dd875dac24609441423c7c76c2ae4719a8/dodo-margin-trading-contracts/contracts/marginTrading/MarginTradingFactory.sol#L267

## Tool used

Manual Review

## Recommendation

Let user give spending allowance to the TradingFactory contract and TradingMargin contract instead of DODOApprove contract!


