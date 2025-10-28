# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Size is a credit marketplace protocol with unified liquidity across maturities. The repository contains two main projects:
- `size-solidity/`: Core Size protocol smart contracts (Solidity)
- `very-liquid-vaults/`: Vault integration contracts

The Size protocol is deployed on Ethereum mainnet and Base, and has undergone multiple security audits (see audits/ directory).

## Architecture

### Core Design Pattern (FREI-PI)

The Size v2 architecture follows the dYdX v2 pattern with these key principles:

1. **Single Entry Point**: `Size.sol` is the main contract behind a UUPS-Upgradeable proxy
2. **External Libraries**: All business logic is in external libraries called via `delegatecall`
3. **Shared State**: A single `State storage` variable is passed to all library functions
4. **Consistent Flow**: All user-facing functions follow the pattern:
   ```solidity
   state.validateFunction(params);
   state.executeFunction(params);
   state.validateInvariant(params);
   ```

This pattern overcomes EIP-170's 24kb contract size limit and maintains protocol invariants after each interaction.

### Directory Structure

- `src/market/`: Core Size market contracts and libraries
  - `Size.sol`: Main entry point contract
  - `SizeView.sol`: View functions for reading protocol state
  - `SizeStorage.sol`: State storage definitions
  - `libraries/actions/`: Individual action implementations (Deposit, Withdraw, Borrow, Liquidate, etc.)
  - `libraries/`: Core logic (AccountingLibrary, LoanLibrary, RiskLibrary, YieldCurveLibrary, etc.)
- `src/factory/`: SizeFactory for deploying and managing multiple markets
- `src/collections/`: CollectionsManager for curator-based rate provision (v1.8+)
- `src/oracle/`: Price feed implementations (Chainlink, Uniswap TWAP, Morpho, Pendle)
- `src/helpers/`: Utility contracts (ReentrancyGuard with view modifier support)

### Key Contracts

1. **Size.sol**: Main protocol contract with all user-facing functions
2. **SizeFactory.sol**: Factory for deploying markets with shared liquidity across markets (v1.5+)
3. **CollectionsManager.sol**: Manages collections, curators, and rate providers (v1.8+)
4. **NonTransferrableRebasingTokenVault**: "Vault of vaults" for deposit tokens (v1.8+, replaces v1.5's NonTransferrableScaledTokenV1_5)

### Token System

- **Deposit Tokens**: Users deposit underlying tokens (USDC, WETH) and receive 1:1 non-transferrable deposit tokens (szaUSDC, szWETH)
- **No Native Ether**: Only wrapped ether (WETH) to prevent donation/reentrancy attacks
- **Debt Tokens**: szDebt tracks borrower obligations (same decimals as borrow token)
- **Vault System (v1.8)**: Users can select different yield vaults (Aave by default, or whitelisted ERC4626 vaults)

## Building and Testing

### Setup
```bash
cd size-solidity
forge install  # Install dependencies
```

### Build
```bash
forge build
forge build --sizes  # Check contract sizes
```

### Tests
```bash
# Unit tests
forge test

# Fork tests (requires API_KEY_ALCHEMY in .env)
source .env
FOUNDRY_PROFILE=fork forge test

# Run specific test
forge test --match-test testName

# Verbose output
forge test -vvv

# Run single test file
forge test --match-path test/path/to/Test.sol
```

### Invariant Testing
```bash
# Foundry invariants
forge test --match-contract FoundryTester

# Echidna
yarn echidna-property  # Property mode
yarn echidna-assertion # Assertion mode
yarn echidna-coverage  # View coverage report

# Onchain fuzzing
source .env
FOUNDRY_PROFILE=fork FOUNDRY_INVARIANT_RUNS=0 FOUNDRY_INVARIANT_DEPTH=0 forge test --mc FoundryForkTester -vvvvv --ffi
```

### Formal Verification
```bash
# Halmos (symbolic execution for Math.binarySearch)
halmos --match-contract Math
# Or run for different loop depths:
for i in {0..5}; do halmos --loop $i; done
```

### Linting and Formatting
```bash
npm install           # Install dependencies
npm run solhint      # Lint Solidity code
forge fmt            # Format code
forge fmt --check    # Check formatting
```

### Coverage
```bash
forge coverage --no-match-coverage "(script|test|deprecated)" --report lcov
genhtml lcov.info -o report --branch-coverage
```

## Deployment

### Environment Setup
Create `.env` file with:
```bash
API_KEY_ALCHEMY=<Your Alchemy API Key>
API_KEY_ETHERSCAN=<Your Etherscan API Key>
DEPLOYER_ADDRESS=<Deployer's Ethereum Address>
DEPLOYER_ACCOUNT=<Name of Deployer's Account in Foundry>
OWNER=<Owner's Address>
FEE_RECIPIENT=<Fee Recipient's Address>
NETWORK_CONFIGURATION=<Network Configuration>
RPC_URL=<Network Name>
```

### Deploy Market
```bash
source .env
export NETWORK_CONFIGURATION=base-production-weth-usdc
forge script script/Deploy.s.sol --rpc-url $RPC_URL --gas-limit 30000000 --sender $DEPLOYER_ADDRESS --account $DEPLOYER_ACCOUNT --ffi --verify -vvvvv
```

### Upgrade Contract
```bash
source .env
forge script script/Upgrade.s.sol --rpc-url $RPC_URL --gas-limit 30000000 --sender $DEPLOYER_ADDRESS --account $DEPLOYER_ACCOUNT --ffi --verify -vvvvv
```

## Key Protocol Features

### Collections, Curators, and Rate Providers (v1.8)

The v1.8 release introduced a sophisticated system for yield curve management:

- **Collection**: A set of markets grouped under a curator
- **Curator**: Defines rate providers (RPs) for each market in their collection
- **Rate Provider**: Accounts that set yield curves and compete on pricing credit
- Users subscribe to collections and inherit rate provider curves
- Users can override with their own curves at the market level
- Multiple collections and rate providers per market are supported

### Authorization System (v1.7+)

Users can authorize operator accounts to perform actions on their behalf via `SizeFactory.setAuthorization()`. This enables:
- Automated refinancing and stop-loss strategies
- One-click leverage via looping contracts
- Keeper bots for complex operations

**Security Note**: Authorization is powerful but risky. Only authorize trusted contracts/wallets. Best practice: authorize at start of multicall, revoke at end.

### Custom Vaults (v1.8+)

Users can select different yield-bearing vaults for their borrow token deposits:
- Default: Aave v3
- Custom: Admin-whitelisted ERC4626-compatible vaults
- Set via `setUserConfiguration(vault: address)`
- Adapter pattern: `AaveAdapter` and `ERC4626Adapter` implement `IAdapter`
- Vault compromises are isolated to users of that vault

### Access Control

Four role levels (checked on market first, then SizeFactory):
- `DEFAULT_ADMIN_ROLE`: Admin operations, upgrades, config updates
- `PAUSER_ROLE`: Pause/unpause protocol
- `KEEPER_ROLE`: Execute keeper functions like liquidateWithReplacement
- `BORROW_RATE_UPDATER_ROLE`: Update variable borrow rate oracle

### Multicall

All user-facing functions have the `payable` modifier to support multicall with ether deposits. The protocol always uses `address(this).balance` for wrapping ether, crediting any forcibly-sent ether to the depositor.

## Mathematical Conventions

- **Explicit Rounding**: All math uses `mulDivUp` or `mulDivDown` from Solady's FixedPointMathLib
- **Rounding Direction**: Favors maker (passive party) in taker-maker operations
- **Decimals**:
  - USDC/aUSDC: 6 decimals
  - WETH/szETH: 18 decimals
  - szDebt: same as borrow token
  - Price feeds: 18 decimals
  - Percentages: 18 decimals (e.g., 150% = 1500000000000000000)

## Protocol Invariants

Invariants are defined in `test/invariants/PropertiesSpecifications.sol` and include:
- **SOLVENCY_01**: SUM(outstanding credit) == SUM(outstanding debt)
- **TOKENS_01**: Sum of collateral deposit tokens equals underlying collateral
- **UNDERWATER_01**: Users cannot make operations leaving others underwater
- **VAULTS_01**: SUM(balanceOf) <= totalSupply()
- See PropertiesSpecifications.sol for complete list

## Security Considerations

### Known Limitations
- No support for rebasing or fee-on-transfer tokens
- Only IERC20Metadata-compliant, pre-vetted tokens
- Trusted roles: owner, KEEPER_ROLE, PAUSER_ROLE, BORROW_RATE_UPDATER_ROLE
- Oracle risks: Chainlink failures may cause incorrect liquidations
- Pause risks: Price changes during pause may cause unforeseen liquidations
- USDC blacklisting may prevent withdrawals
- Vault failures (caps, low liquidity) may prevent deposit/withdraw
- LiquidateWithReplacement may not be available for large debt positions

### Audits
Multiple audits from Spearbit, Code4rena, Cantina, Custodia Security, ChainDefenders, Omniscia, and Hashlock. See `audits/` directory and README.md for details. Bug bounty program available at Cantina.

### Reentrancy Protection
- Uses custom `ReentrancyGuardUpgradeableWithViewModifier` to support view functions
- Important: v1.8 removed reentrancy guard (see commented line in Size.sol:103)

## Development Notes

- **Contract Size Limit**: Architecture specifically designed to handle EIP-170's 24kb limit via external libraries
- **Upgradeable**: UUPS proxy pattern allows upgrades via admin role
- **Version History**: Currently v1.8 (see README for version history and breaking changes)
- **Collections Breaking Changes**: v1.8 changed copy trading behavior - user curves no longer overridden by rate providers
- **Solidity Version**: 0.8.23 with Shanghai EVM version
- **Optimizer**: Enabled with 200 runs

## Useful Scripts

See `script/` directory for deployment and interaction scripts:
- `Deploy.s.sol`: Main deployment script
- `Upgrade.s.sol`: Upgrade existing deployment
- `Networks.sol`: Network configurations
- Various operation scripts: BuyCreditMarket.s.sol, SellCreditLimit.s.sol, Liquidate.s.sol, etc.
- Safe multisig proposal scripts: ProposeSafeTx*.s.sol

## CI/CD

GitHub Actions workflow (`.github/workflows/ci.yml`) runs:
1. Contract size checks
2. Format checks (forge fmt)
3. Unit tests (forge test)
4. Fork tests with mainnet state
5. Slither static analysis
6. Solhint linting
7. Invariant testing (Echidna property/assertion modes, Medusa)
8. Halmos formal verification
9. Coverage reporting (Coveralls)

## Documentation

- **Whitepaper**: https://docs.size.cash/
- **Deployments**: See `deployments/` directory for mainnet and Base addresses
- **Version Interfaces**: Multiple version interfaces in `src/market/interfaces/` (v1.7, v1.8)
