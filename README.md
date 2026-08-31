## Smart Contract Security Research

Security research and audit work focused on Solidity, DeFi protocols, and EVM smart contracts.

## Focus Areas

- DeFi security
- Lending protocols
- Vaults and ERC-4626
- Protocol accounting
- State-transition vulnerabilities
- Economic attacks
- Access control
- Oracle and pricing risks
- Upgradeable smart contracts
- Solidity / Foundry security testing

## Selected Research

**Midnight — Multicall Bad-Debt Escape**

**Severity**: Medium
**Category**: Accounting / State Transition / Economic

A lender can atomically withdraw before bad debt is realized and subsequently trigger liquidation in the same transaction, allowing the lender to avoid its proportional share of the loss.

"Read the full finding" (findings/midnight/M-01-multicall-bad-debt-escape.md)

**Monetrix — Foundation Yield Share**

**Severity**: Low
**Category**: Configuration / Protocol Revenue

Governance can configure the yield distribution such that the foundation's residual share becomes zero, eliminating the protocol's intended foundation yield allocation.

"Read the full finding" (findings/monetrix/L-01-foundation-yield-share.md)

## Methodology

My research process emphasizes understanding the protocol's intended invariants and economic assumptions before attempting to violate them.

I focus on:

1. Understanding protocol architecture and state transitions.
2. Identifying critical invariants and accounting relationships.
3. Constructing adversarial transaction sequences.
4. Testing edge cases and unexpected state combinations.
5. Building reproducible Proofs of Concept.
6. Distinguishing exploitable vulnerabilities from theoretical code smells.

## Tooling

- Solidity
- Foundry
- Git
- EVM security analysis

## Research Philosophy

I don't consider a suspicious code pattern a vulnerability by itself.

The goal is to demonstrate a concrete security property that can be violated, determine the attacker-controlled conditions required, and establish the resulting impact.
