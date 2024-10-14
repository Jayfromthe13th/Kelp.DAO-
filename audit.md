# Kelp Project Audit Report

This document outlines the security audit conducted for the Kelp Project. The report identifies critical vulnerabilities in smart contracts, their impacts, possible exploits, and recommended mitigation steps.

## Contents

1. [Stale Price Arbitrage Exploit Vulnerability](#stale-price-arbitrage-exploit-vulnerability)
2. [NodeDelegator.sol Vulnerability](#nodedelegatorsol-vulnerability)

## Stale Price Arbitrage Exploit Vulnerability

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
```solidity
// Setup: Deploy mock oracles to simulate Chainlink's price updates for assets like rETH, stETH, and cbETH.
address rETHPriceOracle = address(new LRTOracleMock(1.09149e18)); // Initial rETH price set.
address stETHPriceOracle = address(new LRTOracleMock(0.99891e18)); // Initial stETH price set.
address cbETHPriceOracle = address(new LRTOracleMock(1.05407e18)); // Initial cbETH price set.

// Update the LRTOracle to use the mock price oracles for the assets.
LRTOracle(lrtConfig.getContract(LRTConstants.LRT_ORACLE)).updatePriceOracleFor(address(rETH), rETHPriceOracle);
LRTOracle(lrtConfig.getContract(LRTConstants.LRT_ORACLE)).updatePriceOracleFor(address(stETH), stETHPriceOracle);
LRTOracle(lrtConfig.getContract(LRTConstants.LRT_ORACLE)).updatePriceOracleFor(address(cbETH), cbETHPriceOracle);

// Step 1: Alice deposits assets when prices are initially set.
// Alice deposits a large amount of each asset to establish a baseline for the system.
vm.startPrank(alice);
rETH.approve(address(lrtDepositPool), 9950 ether);
lrtDepositPool.depositAsset(rETHAddress, 9950 ether); // Alice deposits 9950 rETH.

stETH.approve(address(lrtDepositPool), 88170 ether);
lrtDepositPool.depositAsset(address(stETH), 88170 ether); // Alice deposits 88170 stETH.

cbETH.approve(address(lrtDepositPool), 1880 ether);
lrtDepositPool.depositAsset(address(cbETH), 1880 ether); // Alice deposits 1880 cbETH.
vm.stopPrank();

// Step 2: Carol deposits assets when prices are still close to the original spot prices.
// This represents a fair deposit scenario with no significant price changes.
vm.startPrank(carol);
uint256 carolBalanceBefore = rseth.balanceOf(address(carol)); // Carol's initial rsETH balance.

rETH.approve(address(lrtDepositPool), 100 ether);
lrtDepositPool.depositAsset(address(rETH), 100 ether); // Carol deposits 100 rETH.

uint256 carolBalanceAfter = rseth.balanceOf(address(carol)); // Carol's new rsETH balance after deposit.
vm.stopPrank();

// Step 3: Simulate price manipulation by manually adjusting the mock oracle prices.
// This simulates a scenario where real market prices have shifted, but Chainlink's updates could be delayed.
uint256 rETHNewPrice = uint256(LRTOracleMock(rETHPriceOracle).getAssetPrice(address(0))) * 102 / 100; // +2% increase for rETH.
uint256 stETHNewPrice = uint256(LRTOracleMock(stETHPriceOracle).getAssetPrice(address(0))) * 995 / 1000; // -0.5% decrease for stETH.
uint256 cbETHNewPrice = uint256(LRTOracleMock(cbETHPriceOracle).getAssetPrice(address(0))) * 99 / 100; // -1% decrease for cbETH.

LRTOracleMock(rETHPriceOracle).submitNewAssetPrice(rETHNewPrice);
LRTOracleMock(stETHPriceOracle).submitNewAssetPrice(stETHNewPrice);
LRTOracleMock(cbETHPriceOracle).submitNewAssetPrice(cbETHNewPrice);

// Step 4: Bob deposits assets after the price changes to demonstrate arbitrage opportunities.
// The price changes allow Bob to potentially mint more rsETH than Carol for the same amount of deposit.
vm.startPrank(bob);
uint256 bobBalanceBefore = rseth.balanceOf(address(bob)); // Bob's initial rsETH balance.

rETH.approve(address(lrtDepositPool), 100 ether);
lrtDepositPool.depositAsset(address(rETH), 100 ether); // Bob deposits 100 rETH after the price change.

uint256 bobBalanceAfter = rseth.balanceOf(address(bob)); // Bob's new rsETH balance after deposit.
vm.stopPrank();

// Assertions to demonstrate the arbitrage opportunity:
// Bob's rsETH balance should be higher than Carol's due to the updated price, despite both depositing the same amount.
assertGt(bobBalanceAfter, carolBalanceAfter, "Bob minted more rsETH due to favorable price change");
```

Explanation:
- Step 1: Alice deposits large amounts when prices are initially set. This sets up a baseline for the system.
- Step 2: Carol deposits when prices are still close to their original values, resulting in a fair rsETH minting.
- Step 3: Prices are manually adjusted to simulate market changes. This reflects how real-world price updates could occur due to market volatility.
- Step 4: Bob deposits after the price changes, illustrating that due to the new prices, Bob mints more rsETH than Carol for the same deposit amount, demonstrating the risk of arbitrage if oracle updates lag behind real market conditions.


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

