# Kelp Project Audit Report

This document outlines the security audit conducted for the Kelp Project. The report identifies critical vulnerabilities in smart contracts, their impacts, possible exploits, and recommended mitigation steps.

## Contents

1. [ChainlinkPriceOracle.sol Vulnerability](#chainlinkpriceoraclesol-vulnerability)
2. [NodeDelegator.sol Vulnerability](#nodedelegatorsol-vulnerability)

## ChainlinkPriceOracle.sol Vulnerability

*Severity: High*

### Description

- **File:** [ChainlinkPriceOracle.sol](https://github.com/code-423n4/2023-11-kelp/blob/main/src/oracles/ChainlinkPriceOracle.sol)
- **Function:** `getAssetPrice`
- **Lines:** [37-38](https://github.com/code-423n4/2023-11-kelp/blob/main/src/oracles/ChainlinkPriceOracle.sol#L37-L38)

### Impact

The absence of a Chainlink heartbeat check in the `getAssetPrice` function can lead to:
- Reliance on stale or outdated price data, risking incorrect operational decisions.
- Exposure to potential price feed manipulation by malicious actors.

### Proof of Concept

In a simulated scenario:
- A user deposits Ethereum (ETH) to receive RSETH tokens, where pricing is determined by the LRTOracle contract utilizing the `getAssetPrice` function.
- The Chainlink oracle responsible for updating the LST/ETH price goes offline, causing the system to freeze the last recorded price.
- Despite significant market value changes, the frozen price remains in use, leading to transaction inaccuracies.

### Recommended Mitigation

Modify `getAssetPrice` to include a heartbeat check by integrating the `latestRoundData` function to confirm data freshness.

**Example Code:**
```solidity
function getAssetPrice(address asset) external view onlySupportedAsset(asset) returns (uint256) {
    (,int price,,uint timeStamp,) = AggregatorInterface(assetPriceFeed[asset]).latestRoundData();
    require(block.timestamp - timeStamp < acceptableDelay, "Price data is stale");
    return uint(price);
}

```

# NodeDelegator.sol Vulnerability

**Severity: Medium**

## Description

- **File:** NodeDelegator.sol
- **Functions:** General strategy management
- **Lines:** [59](https://github.com/link-to-repo/NodeDelegator.sol#L59), [122](https://github.com/link-to-repo/NodeDelegator.sol#L122)

## Impact

Changing an asset's strategy in `NodeDelegator` without prior withdrawal from the existing strategy may lead to:
- **Loss of funds** due to inability to interact with the old strategy.

## Proof of Concept

In a realistic sequence:
- Assets are deposited into a strategy via `NodeDelegator`.
- A strategy change is initiated without withdrawing the existing funds, rendering the funds inaccessible as the contract no longer references the old strategy.

## Recommended Mitigation

Implement checks to ensure asset withdrawal before strategy updates and modify contracts to keep a registry of all strategies used.

## Example Code

```solidity
function updateAssetStrategy(address asset, address newStrategy) external onlyLRTManager {
    address currentStrategy = assetStrategy(asset);
    require(IStrategy(currentStrategy).balanceOf(address(this)) == 0, "Funds still in current strategy");
    _assetStrategies[asset] = newStrategy;
}

mapping(address => address[]) private _assetStrategies;

function getAssetStrategies(address asset) external view returns (address[] memory) {
    return _assetStrategies[asset];
}

