---
kip: 86
title: Public and decentralized keeper operations for conditional orders
status: Draft
created: 2023-06-17
author: 0xdapper
section: <#>
---

## Simple Summary

Update the `Account` contract implementation to make `executeConditionalOrder`
public instead of it being gated and callable only by gelato executors i.e.
[remove the `onlyOps` modifier](https://github.com/Kwenta/margin-manager/blob/main/src/Account.sol#L742-L746).

## Abstract

This change will allow anyone to run keeper and execute the conditional
orders for kwenta smart margin account users and decentralize keeper
infrastructure so it is not reliant on gelato network. It will be equivalent
to how synthetix has public keeper operations with keeper rewards to incentivize
the actions to take place at the earliest.

## Motivation

Because of gelato's design, the best execution we can get right now for conditional orders
are suboptimal. Right now the gelato network polls every registered tasks for checking
if it is ready for execution which in turn checks if the
[chainlink price of giving market is meeting the conditional order conditions or not](https://github.com/Kwenta/margin-manager/blob/main/src/Account.sol#L826-L856).
Chainlink obviously updates slowly while order execution on
synthetix is done faster with pyth oracle prices.

It is already possible to expose a function that rather takes pyth signed price update[^1]
as parameter and verify its integrity on chain and trigger the conditional order if it meets
the conditions. With such a function it will automatically also be possible to make the
`executeConditionalOrder` or some other function public and let anyone execute it.

The keepers can be incentivized with some reward for executing the conditional orders as
soon as a pyth price update can meet the condition.

## Specification

Pseudo-code as follows:

```solidity
    function executeConditionalOrder(
        uint256 _conditionalOrderId,
        bytes[] calldata updateData,
        bytes32[] priceId,
        uint minPublishTime,
        uint maxPublishTime
    ) external isAccountExecutionEnabled {
        ...
        // verify the signature and parse the price
        PythStructs.Price[] memory priceUpdates = pyth.parsePriceFeedUpdates(...);
        int price = priceUpdates[0].price;
        require(_validConditionalOrder(_conditionalOrderId, uint(price)));
        ...
    }

    function _validConditionalOrder(uint256 _conditionalOrderId, uint price)
        internal
        view
        returns (bool)
    {
        ConditionalOrder memory conditionalOrder =
            getConditionalOrder(_conditionalOrderId);

        // return false if market is paused
        try SYSTEM_STATUS.requireFuturesMarketActive(conditionalOrder.marketKey)
        {} catch {
            return false;
        }

        // check if markets satisfy specific order type
        if (
            conditionalOrder.conditionalOrderType == ConditionalOrderTypes.LIMIT
        ) {
            return _validLimitOrder(conditionalOrder, price);
        } else if (
            conditionalOrder.conditionalOrderType == ConditionalOrderTypes.STOP
        ) {
            return _validStopOrder(conditionalOrder, price);
        }

        // unknown order type
        return false;
    }
```

[^1]: [signed price update on-chain verification](https://docs.pyth.network/benchmarks#on-chain-contracts)
