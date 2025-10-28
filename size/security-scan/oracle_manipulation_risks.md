# Oracle Manipulation Risk Assessment

## Executive Summary
Both the Size protocol and Very Liquid Vaults rely on external price feeds, creating significant oracle manipulation risks. These risks are particularly concerning in the context of flash loan attacks, where attackers can temporarily manipulate oracle prices to trigger liquidations or exploit pricing inefficiencies.

## Detailed Analysis

### 1. Size Protocol Oracle Risks

**Location**: `IPriceFeed` integration in Size protocol

**Vulnerability**: 
- The protocol relies on external price feeds that can be manipulated through flash loans on underlying DEXs
- The `collateralRatio()` function uses `state.oracle.priceFeed.getPrice()` directly
- No time-weighted averaging or other protection mechanisms are evident

**Attack Vector**:
1. Flash loan large amounts of collateral or borrow tokens
2. Manipulate DEX prices to affect oracle feed
3. Trigger liquidations or other state changes based on false prices
4. Profit from the resulting market movements

**Risk Level**: HIGH

### 2. Very Liquid Vaults Oracle Risks

**Location**: Potential oracle dependencies in strategy valuations

**Vulnerability**:
- Strategies may rely on external price feeds for asset valuations
- The `totalAssets()` calculation depends on strategy valuations which may use oracles
- No explicit oracle protection mechanisms are evident

**Attack Vector**:
1. Manipulate oracle prices through flash loans
2. Affect strategy valuations
3. Manipulate vault share prices
4. Profit from the manipulation

**Risk Level**: MEDIUM

### 3. Common Oracle Vulnerabilities

**TWAP Manipulation**:
- Even if TWAP oracles are used, they can still be manipulated with large flash loans over extended periods
- The longer the TWAP period, the more capital required but the more reliable the manipulation

**Centralization Risks**:
- Both protocols appear to rely on single oracle sources (e.g., Chainlink)
- If these oracles are compromised or report incorrect prices, it affects both protocols
- No fallback mechanisms are clearly implemented

**Staleness Issues**:
- Oracle prices may become stale, leading to incorrect calculations
- No staleness checks are evident in the code

## Recommendations

1. **Implement TWAP Oracles**: Use Time-Weighted Average Price oracles with longer time windows to make manipulation more difficult and expensive.

2. **Add Multiple Oracle Sources**: Implement multiple oracle sources and use median or weighted average pricing to reduce single points of failure.

3. **Add Staleness Checks**: Implement staleness checks for all oracle prices to prevent operations with outdated data.

4. **Implement Circuit Breakers**: Add circuit breakers that pause operations when oracle prices move beyond certain thresholds.

5. **Add Oracle Validation**: Implement validation mechanisms to detect anomalous oracle price movements.

6. **Regular Oracle Audits**: Conduct regular audits of oracle integrations and dependencies.