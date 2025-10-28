# Security Scan Plan for Size Credit Projects

## Overview
This document outlines the security analysis plan for the Size protocol and Very Liquid Vaults projects. The analysis will focus on identifying vulnerabilities, particularly those related to flash loan attacks, cross-protocol exploits, and economic attack vectors.

## Scope
- Size Solidity smart contracts (commit: 739250c26be314b0d74e670297a344b02be625d0)
- Very Liquid Vaults smart contracts (commit: ba41d98fc100f385d3d7010d4264cd8c7a4faf71)

## Analysis Areas

### 1. Flash Loan Attack Surfaces
- Direct integration points with external tokens
- Collateral ratio manipulation vulnerabilities
- Liquidation mechanism exploitation
- Vault share price manipulation
- Strategy rebalancing vulnerabilities

### 2. Cross-Protocol Attack Vectors
- Aave integration vulnerabilities
- Oracle price manipulation across protocols
- Liquidity pool manipulation
- Feedback loop exploitation

### 3. Economic Attack Vectors
- Yield curve manipulation
- Liquidation economics
- Fee mechanism vulnerabilities
- Market order exploitation

### 4. Core Security Issues
- Access control vulnerabilities
- Reentrancy protection
- Oracle manipulation risks
- Storage layout hazards
- Upgrade-related issues
- Mathematical precision errors

## Methodology
1. Static code analysis
2. Dynamic testing with Foundry
3. Manual review of critical functions
4. Attack vector modeling
5. Risk assessment and prioritization

## Deliverables
1. Detailed vulnerability report with risk levels
2. Remediation recommendations
3. Security monitoring implementation plan
4. TODO comment integration for ongoing tracking

## Timeline
- Initial analysis: 3 days
- Detailed vulnerability assessment: 5 days
- Remediation planning: 2 days
- Final reporting: 1 day