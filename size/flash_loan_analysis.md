# Flash Loan Attack Surface Analysis: Size Protocol & Very Liquid Vaults

## Executive Summary

This document provides a comprehensive analysis of flash loan attack surfaces in both the Size protocol and Very Liquid Vaults projects. Both projects have unique vulnerabilities and attack vectors due to their different architectural approaches.

## Project Overview

### Size Protocol
- A fixed-term lending protocol that allows users to create debt and credit positions with specific terms
- Uses Aave v3 as the underlying variable rate pool for borrow tokens
- Has a complex order book system for matching lenders and borrowers
- Features liquidation mechanisms for undercollateralized positions

### Very Liquid Vaults
- A vault system that distributes assets across multiple strategies
- Includes strategies for Aave, ERC4626-compliant vaults, and cash positions
- Implements performance fees and rebalancing mechanisms
- Uses role-based access control for management functions

## 1. Direct Flash Loan Integration Points

### Size Protocol

#### 1.1 Deposit/Withdraw Functions
The `deposit` and `withdraw` functions in the Size contract are vulnerable to flash loan attacks because:

- They interact with external tokens (collateral and borrow tokens)
- They update internal balances that affect collateral ratios
- A flash loan could temporarily inflate balances to manipulate liquidation thresholds

**Attack Vector**: 
1. Borrow a large amount of underlying tokens via flash loan
2. Deposit these tokens into Size to temporarily increase collateral ratio
3. Execute operations that require higher collateral ratios (e.g., borrow more)
4. Withdraw tokens and repay flash loan

#### 1.2 Liquidation Functions
The `liquidate` function is particularly vulnerable because:

- It calculates liquidation profitability based on current collateral ratios
- Flash loan manipulation of oracle prices or token balances could trigger incorrect liquidations
- Liquidators might be incentivized to manipulate prices to create profitable liquidation opportunities

**Attack Vector**:
1. Manipulate the oracle price feed using flash loans on the underlying assets
2. Cause undercollateralized positions to become liquidatable
3. Execute liquidations at favorable rates
4. Repay flash loan and keep profits

#### 1.3 Market Functions
Functions like `buyCreditMarket`, `sellCreditMarket`, and `repay` can be exploited:

- These functions involve complex calculations based on current market conditions
- Flash loan manipulation can affect the perceived value of credit positions
- APR calculations and tenor-based pricing can be manipulated

### Very Liquid Vaults

#### 1.4 Vault Deposit/Withdraw Functions
The `VeryLiquidVault` contract inherits from `ERC4626Upgradeable`:

- Deposit and withdraw functions affect the vault's share price (exchange rate)
- Flash loans can manipulate the totalAssets() calculation by affecting underlying strategy balances
- This can lead to share price manipulation during deposits/withdrawals

**Attack Vector**:
1. Flash loan large amounts and deposit into underlying strategies
2. Cause vault share price manipulation
3. Execute profitable deposits/withdrawals at the expense of other users
4. Withdraw and repay flash loan

#### 1.5 Strategy Rebalancing
The `rebalance` function in `VeryLiquidVault` is vulnerable:

- It moves assets between strategies based on current valuations
- Flash loan manipulation can affect the perceived value of strategies
- Slippage protection might not be sufficient during flash loan events

## 2. Indirect Flash Loan Exploitation Vectors

### 2.1 Oracle Price Manipulation
Both projects rely on external price feeds:

- Size Protocol uses `IPriceFeed` for collateral ratio calculations
- Oracle prices can be manipulated through flash loan attacks on underlying DEXs
- This affects liquidation thresholds and risk calculations

**Attack Scenario**:
1. Flash loan large amounts of collateral or borrow tokens
2. Manipulate prices on DEXs to affect oracle feeds
3. Cause liquidations or other state changes in Size protocol
4. Profit from the resulting market inefficiencies

### 2.2 Aave Integration Vulnerabilities
Both projects integrate with Aave v3:

- Size deposits borrow tokens into Aave for lenders
- Very Liquid Vaults has Aave strategy vaults
- Flash loan attacks on Aave can affect both protocols' operations

### 2.3 Collateral Ratio Manipulation
Size Protocol's risk management depends on collateral ratios:

- Formula: `(collateral * price) / debt`
- Flash loans can temporarily inflate collateral or deflation debt
- This affects liquidation thresholds and borrowing capacity

## 3. Multi-Contract Attack Compositions

### 3.1 Cross-Protocol Manipulation
Attackers can combine attacks on both protocols:

1. Manipulate Very Liquid Vault share prices
2. Affect underlying Aave positions that Size protocol relies on
3. Create profitable opportunities across both protocols

### 3.2 Oracle & Strategy Manipulation Combo
1. Manipulate oracle prices through DEX flash loans
2. Trigger rebalancing in Very Liquid Vaults
3. Exploit the rebalancing at manipulated prices
4. Cause liquidations in Size protocol at manipulated ratios

### 3.3 Liquidation Front-Running
1. Monitor pending transactions for large deposits/withdrawals
2. Execute flash loan to manipulate oracle prices
3. Trigger liquidations that become profitable due to manipulation
4. Execute liquidations and repay flash loan

## 4. Cross-Protocol Value Extraction

### 4.1 Yield Arbitrage
- Size protocol offers fixed-term lending rates
- Very Liquid Vaults can invest in various yield sources
- Flash loans can be used to exploit temporary yield differentials

### 4.2 Share Price Manipulation
- Very Liquid Vaults use share-based accounting
- Flash loans can manipulate share prices for profitable entry/exit
- Affects all vault participants

### 4.3 Fee Extraction
- Both protocols charge various fees
- Manipulation can force fee payments at advantageous rates
- Performance fees in Very Liquid Vaults can be gamed

## 5. Liquidity Pool Manipulation

### 5.1 DEX Manipulation
- Both protocols interact with external DEXs for token swaps
- Flash loans can manipulate DEX reserves
- Affects oracle prices and swap rates

### 5.2 Aave Pool Manipulation
- Both protocols interact with Aave lending pools
- Flash loans can affect Aave's interest rates
- Impacts Size protocol's variable rate calculations

### 5.3 Internal Liquidity Exploitation
- Size protocol has internal order book liquidity
- Very Liquid Vaults have strategy-based liquidity
- Flash loans can exploit temporary liquidity imbalances

## 6. Market Impact Amplification

### 6.1 Cascading Liquidations
- Oracle manipulation can trigger multiple liquidations
- Creates market panic and further price drops
- Amplifies the impact of the initial flash loan

### 6.2 Feedback Loops
- Liquidations affect underlying token prices
- Price changes trigger more liquidations
- Very Liquid Vault rebalancing amplifies effects

### 6.3 Systemic Risk
- Both protocols are interconnected through Aave
- Flash loan attacks on one can affect the other
- Potential for protocol-wide instability

## Detailed Attack Scenarios

### Scenario 1: Collateral Ratio Manipulation Attack

**Target**: Size Protocol liquidation mechanism
**Capital Required**: Large flash loan of collateral token
**Steps**:
1. Obtain flash loan of collateral token (e.g., WETH)
2. Deposit large amount to target user's account to inflate collateral ratio
3. Manipulate oracle to show favorable price
4. Execute profitable liquidations of other users' positions
5. Withdraw collateral and repay flash loan

**Profit Calculation**:
- Profit = Liquidation rewards + liquidation fees - flash loan fees - gas
- Risk: Oracle manipulation may be detected; gas costs high

### Scenario 2: Vault Share Price Manipulation

**Target**: Very Liquid Vault deposit/withdraw mechanism
**Capital Required**: Large flash loan of vault's underlying asset
**Steps**:
1. Flash loan vault's underlying asset
2. Deposit large amount to inflate totalAssets
3. Execute withdrawal at inflated share price
4. Repay flash loan

**Profit Calculation**:
- Profit = (manipulated share price - actual share price) * shares withdrawn - flash loan fees - gas

### Scenario 3: Cross-Protocol Arbitrage

**Target**: Both protocols simultaneously
**Capital Required**: Flash loans on multiple assets
**Steps**:
1. Manipulate Aave interest rates via flash loans
2. Affect Size protocol's variable rate calculations
3. Execute profitable trades in Size protocol
4. Simultaneously manipulate Very Liquid Vault strategies
5. Profit from both protocols

## Mitigation Strategies

### For Size Protocol:
1. Implement TWAP oracles with longer time windows
2. Add minimum time delays between deposits and risk-sensitive operations
3. Implement circuit breakers for extreme market conditions
4. Use more sophisticated liquidation triggers
5. Add flash loan resistant pricing mechanisms

### For Very Liquid Vaults:
1. Implement time-weighted deposit/withdraw pricing
2. Add minimum time delays for vault operations
3. Implement slippage protection for rebalancing
4. Use more sophisticated rebalancing triggers
5. Add monitoring for unusual deposit/withdraw patterns

### General Recommendations:
1. Implement MEV protection mechanisms
2. Add transaction monitoring and anomaly detection
3. Use multiple oracle sources
4. Implement economic incentive alignment
5. Regular security audits and testing