# Technical Flash Loan Attack Analysis: Size Protocol & Very Liquid Vaults

## 1. Direct Flash Loan Attack Vectors

### 1.1 Size Protocol - Collateral Ratio Manipulation

**Vulnerable Function**: `RiskLibrary.collateralRatio()`

```solidity
function collateralRatio(State storage state, address account) public view returns (uint256) {
    uint256 collateral = state.data.collateralToken.balanceOf(account);
    uint256 debt = state.data.debtToken.balanceOf(account);
    uint256 price = state.oracle.priceFeed.getPrice();

    if (debt != 0) {
        return Math.mulDivDown(
            collateral * 10 ** state.data.underlyingBorrowToken.decimals(),
            price,
            debt * 10 ** state.data.underlyingCollateralToken.decimals()
        );
    } else {
        return type(uint256).max;
    }
}
```

**Attack Vector**: 
A flash loan can temporarily increase a user's collateral balance to manipulate their collateral ratio, allowing them to:
- Bypass liquidation thresholds
- Take on more debt than they should be allowed
- Manipulate liquidation opportunities for other users

**Example Attack Sequence**:
```
1. Flash loan large amount of collateral token (e.g., WETH)
2. Deposit collateral to Size protocol to increase balance
3. Collateral ratio increases, allowing new borrowing
4. Borrow additional funds while maintaining apparent safety
5. Withdraw collateral and repay flash loan
6. Position is now overleveraged but temporarily appears safe
```

### 1.2 Size Protocol - Oracle Price Manipulation

**Vulnerable Component**: `IPriceFeed` integration

The protocol relies on external price feeds that can be manipulated through flash loans on underlying DEXs.

**Attack Vector**:
1. Flash loan large amounts of collateral or borrow tokens
2. Manipulate DEX prices to affect oracle feed
3. Trigger liquidations or other state changes based on false prices
4. Profit from the resulting market movements

### 1.3 Very Liquid Vault - Share Price Manipulation

**Vulnerable Function**: `VeryLiquidVault.totalAssets()`

```solidity
function totalAssets() public view virtual override(ERC4626Upgradeable, IERC4626) returns (uint256 total) {
    VeryLiquidVaultStorage storage $ = _getVeryLiquidVaultStorage();
    uint256 length = $._strategies.length;
    for (uint256 i = 0; i < length; ++i) {
        IVault strategy = $._strategies[i];
        uint256 strategyBalance = strategy.balanceOf(address(this));
        // slither-disable-next-line incorrect-equality
        if (strategyBalance == 0) continue;
        total += strategy.convertToAssets(strategyBalance);
    }
}
```

**Attack Vector**:
1. Flash loan assets and deposit into underlying strategies
2. Inflate strategy balances and totalAssets
3. Deposit at inflated share price or withdraw at inflated rate
4. Withdraw and repay flash loan, keeping profit

## 2. Multi-Step Flash Loan Attack Implementation

### 2.1 Cross-Protocol Manipulation Attack

Here's a conceptual implementation of a multi-step attack:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "@src/market/ISize.sol";
import "@src/IVault.sol";

contract FlashLoanAttacker {
    ISize public size;
    IVault public veryLiquidVault;
    
    constructor(address _size, address _veryLiquidVault) {
        size = ISize(_size);
        veryLiquidVault = IVault(_veryLiquidVault);
    }
    
    function attack() external {
        // This would be called from a flash loan provider
        executeAttack();
    }
    
    function executeAttack() internal {
        // Step 1: Manipulate oracle through DEX manipulation
        // (Conceptual - actual implementation would depend on specific oracle)
        
        // Step 2: Deposit large amounts to inflate collateral ratios
        // This would allow borrowing more than normally permitted
        
        // Step 3: Execute profitable liquidations of other positions
        // using temporarily manipulated ratios
        
        // Step 4: Manipulate Very Liquid Vault share prices
        // by affecting underlying strategies
        
        // Step 5: Profit extraction and repayment
    }
}
```

### 2.2 Liquidation Front-Running Attack

```solidity
// Attack that monitors for profitable liquidation opportunities
// and manipulates them to become even more profitable

function manipulateLiquidation(uint256 debtPositionId) external {
    // 1. Check if target position is near liquidation threshold
    // 2. Flash loan to manipulate oracle price downward
    // 3. This makes the position appear more undercollateralized
    // 4. Execute liquidation at better terms
    // 5. Repay flash loan
    
    LiquidateParams memory params = LiquidateParams({
        debtPositionId: debtPositionId,
        minimumCollateralProfit: 0,  // Minimal constraint to increase success
        deadline: block.timestamp + 1 // Immediate execution
    });
    
    size.liquidate(params);
}
```

## 3. Profitability Calculations

### 3.1 Collateral Ratio Manipulation Profit

**Scenario**: Manipulate a position to avoid liquidation and maintain borrowing power

- Flash loan cost: 0.09% of borrowed amount
- Gas costs: ~0.1 ETH for complex transaction
- Liquidation reward avoided: Collateral value * liquidationRewardPercent
- Additional borrowing capacity: Debt increase * collateralRatio

**Break-even calculation**:
```
Profit = Liquidation reward avoided + Additional borrowing value - Flash loan fees - Gas costs
```

### 3.2 Share Price Manipulation Profit

**Scenario**: Manipulate Very Liquid Vault share price for profitable entry/exit

- Manipulation cost: Flash loan fees + slippage
- Profit: (Manipulated price - Fair price) * transaction amount
- Gas costs: ~0.2 ETH for complex rebalancing

**Example calculation**:
```
If vault has $1M in assets and 100k shares (NAV = $10/share)
Flash loan adds $500k temporarily, making NAV = $15/share
Attacker withdraws 10k shares worth $150k at $100k fair value
Profit = $50k - flash loan fees - gas
```

## 4. Specific Attack Scenarios with Code

### 4.1 Oracle Manipulation via Aave Integration

Since Size protocol uses Aave for variable rate pools, oracle manipulation can affect the entire system:

```solidity
// Manipulation through Aave supply/demand
function manipulateAaveRate() external {
    // 1. Flash loan large amount of borrow token
    // 2. Supply to Aave to manipulate borrow rates
    // 3. This affects Size protocol's variablePoolBorrowRate
    // 4. Create arbitrage opportunities in Size market functions
    // 5. Withdraw from Aave and repay flash loan
}
```

### 4.2 Strategy Rebalancing Manipulation

Very Liquid Vaults can be attacked through their rebalancing mechanism:

```solidity
function manipulateRebalance(IVault strategyFrom, IVault strategyTo, uint256 amount) external {
    // 1. Flash loan to inflate one strategy's balance
    // 2. Trigger rebalancing between strategies
    // 3. Execute rebalance at manipulated prices
    // 4. Profit from the price differential
    // 5. Repay flash loan
    
    veryLiquidVault.rebalance(strategyFrom, strategyTo, amount, 0.01e18); // 1% max slippage
}
```

## 5. Advanced Multi-Contract Attack Combinations

### 5.1 Feedback Loop Attack

Create a feedback loop where manipulation of one protocol affects the other:

1. Manipulate Very Liquid Vault strategy that invests in Aave
2. This affects Aave's interest rates
3. Size protocol's variable rate calculations are affected
4. Create liquidation opportunities in Size
5. Use liquidation profits to further manipulate Very Liquid Vault
6. Repeat to amplify profits

### 5.2 Cross-Protocol Arbitrage

```solidity
function crossProtocolArbitrage() external {
    // 1. Detect price differential between protocols
    // 2. Use flash loan to exploit the differential
    // 3. Execute on both protocols simultaneously
    // 4. Capture the spread
    
    // Example: Yield differential between Size lending rates 
    // and Very Liquid Vault strategy yields
}
```

## 6. Risk Assessment and Impact Analysis

### 6.1 High-Risk Vectors
1. Oracle price manipulation (highest impact)
2. Collateral ratio gaming
3. Liquidation mechanism manipulation
4. Share price manipulation in vaults

### 6.2 Medium-Risk Vectors
1. Fee extraction through manipulation
2. Cross-protocol feedback loops
3. Strategy rebalancing gaming

### 6.3 Probability Assessment
- High capital requirements limit frequency
- MEV bots may compete for same opportunities
- Oracle manipulation detection improving over time
- Gas costs significant for complex attacks

## 7. Recommendations for Mitigation

### 7.1 For Size Protocol
1. Implement TWAP oracles with longer time windows
2. Add minimum time delays between deposits and risk-sensitive operations
3. Implement circuit breakers for extreme market conditions
4. Use more sophisticated liquidation triggers
5. Add flash loan resistant pricing mechanisms

### 7.2 For Very Liquid Vaults
1. Implement time-weighted deposit/withdraw pricing
2. Add minimum time delays for vault operations
3. Implement slippage protection for rebalancing
4. Use more sophisticated rebalancing triggers
5. Add monitoring for unusual deposit/withdraw patterns