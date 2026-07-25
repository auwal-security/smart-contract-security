## [L-01] Foundation Yield Share Can Be Set to Zero, Resulting In Foundation Yield Share Is Zero
## Description:
In `MonetrixConfig.sol:;foundationYieldBps()` derives the foundation's share as a residual: 10000 - userYieldBps - insuranceYieldBps. But, the setter function allows: userBps + insuranceBps <= 10000. This means governance can configure:- userYieldBps + insuranceYieldBps = 10000 resulting in foundationYieldBps = 0. This contradicts the implied design expectation that the foundation always receives a portion of yield. Even if governance is trusted,mistakes happens.

**Vulnerable code:-**
[MonetrixConfig.sol#L92] (https://github.com/code-423n4/2026-04-monetrix/blob/3d94be1361ca01d959f9165a78f0d75c5657fe3e/src/core/MonetrixConfig.sol#L92-L94)

**Code that allow the issue:-**
[MonetrixConfig.sol#L96] (https://github.com/code-423n4/2026-04-monetrix/blob/3d94be1361ca01d959f9165a78f0d75c5657fe3e/src/core/MonetrixConfig.sol#L96-L97)

## Impact:
Protocol Revenue Loss. When governance set:- userYieldBps + insuranceYieldBps = 10000 (which is legal in code) foundation receives 0% of yield which removes protocol revenue stream and
long-term sustainability funding.

## Proof of Concept:
user = 7000  
insurance = 3000  
foundation = 0
```solidity
function test_submissionValidity() public {
        // governance set the yield Bps
        vm.prank(admin);
        config.setYieldBps(7000, 3000);

        // user deposit and stake
        _deposit(user1, 100000e6);
        _stake(user1, 100000e6);

        // user fund yieldEscrow so that we have yield to distribute
        vm.prank(user1);
        usdc.transfer(address(yieldEscrow), 100000e6);

        // balance of the foundation before the distribution
        uint256 foundationBalBefore = usdc.balanceOf(address(foundation));

        // oparator call distribute yield
        vm.prank(operator);
        vault.distributeYield();
        // balance of the foundation after the distribution
        uint256 foundationBalAfter = usdc.balanceOf(address(foundation));

        // proving the balance before distribution equal after distribution
        assertEq(foundationBalBefore, foundationBalAfter);
        // proving the balance of the foundation is zero
        assertEq(foundationBalAfter, 0);
    }
```
## Recommended Mitigation:  
There are two different way to resolve this.

1. Set foundation share as constant variable :-
    
  `uint256 constant MIN_FOUNDATION_BPS = 1000;`

or 

3. modify the guard so that foundation can't be zaro  
```diff
       function setYieldBps(uint256 _userBps, uint256 _insuranceBps) external onlyGovernor {
-       require(_userBps + _insuranceBps <= 10000, "Config: bps exceed 10000");
+       require(_userBps + _insuranceBps < 10000, "foundation share cannot be zero");
        userYieldBps = _userBps;
        insuranceYieldBps = _insuranceBps;
        emit YieldBpsUpdated(_userBps, _insuranceBps, 10000 - _userBps - _insuranceBps);
    }
```
1. Is highly recommended except if the project want to modify it is revanue later.
