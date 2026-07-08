# Small Reward Dust Can Become Permanently Unclaimable

## Metadata

- **Number:** #848
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 6:50 AM
- **Last updated:** May 31, 2026 at 10:39 PM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Small Reward Dust Can Become Permanently Unclaimable
---
**#M-52**
- Severity: Medium
- Validity: Tested
- Likelihood: High
- Impact: Very small reward balances can advance payout windows without ever incrementing the global reward index, which strands reward inventory permanently inside the vault and makes exact reward accounting impossible for affected reward units.

## Description

Reward accrual uses integer rounding at two layers.

First, `_calculateHandout(...)` rounds down the amount to distribute over the elapsed interval.

Second, `_accrueRewards(...)` only updates `rewardIndex` and `balanceAccounted` if `tokensToHandout != 0`.

The subtle bug is that `payoutLastPaid` advances unconditionally even when `tokensToHandout == 0`.

For tiny dust balances, that means time keeps moving forward but no accounting ever captures the reward unit into `rewardIndex`.

If the dust is small enough to keep rounding to zero on every accrual step, the reward can remain trapped forever. Claimers receive nothing, while the vault still holds the token.

### Reward accrual advances `payoutLastPaid` even when no reward index increment occurs

```solidity
function _accrueRewards(address _rewardToken) internal {
    RewardInfo storage rewardInfo = rewardTrackers[_rewardToken];

    uint256 balanceLastKnown = rewardInfo.balanceLastKnown;
    rewardInfo.balanceLastKnown = IERC20(_rewardToken).balanceOf(address(this)) + rewardInfo.totalClaimed;

    uint256 elapsed = block.timestamp - rewardInfo.payoutLastPaid;
    uint256 unaccountedBalance = balanceLastKnown - rewardInfo.balanceAccounted;
    uint256 tokensToHandout = _calculateHandout(unaccountedBalance, elapsed);

    if (tokensToHandout != 0) {
        uint256 deltaIndex = Math.mulDiv(tokensToHandout, SCALAR * uint256(10 ** decimals()), totalSupply());

        rewardInfo.rewardIndex += deltaIndex;
        rewardInfo.balanceAccounted += tokensToHandout;
    }

    rewardInfo.payoutLastPaid = block.timestamp;
}
```

Source:
- `contracts/staking/StakingVault.sol:433-452`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L433-L452`

This means the payout schedule advances even when the reward unit never becomes claimable.

### Users can only claim what reached `rewardIndex`

```solidity
function _accrueUser(address _user, address _rewardToken) internal {
    ...
    uint256 deltaIndex = rewardInfo.rewardIndex - userRewardTracker.lastRewardIndex;

    if (deltaIndex != 0) {
        uint256 supplierDelta = Math.mulDiv(balanceOf(_user), deltaIndex, uint256(10 ** decimals()) * SCALAR);

        userRewardTracker.accruedRewards += supplierDelta;
        userRewardTracker.lastRewardIndex = rewardInfo.rewardIndex;
    }
}
```

Source:
- `contracts/staking/StakingVault.sol:455-474`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L455-L474`

If `rewardIndex` never moves, no claimer can ever accrue the dust unit.

## Root Cause

The root cause is that the reward schedule treats elapsed time as consumed even when integer rounding prevents any corresponding reward-index progress.

```text
root cause:
- tiny reward balances can round down to zero handout amounts
- rewardIndex and balanceAccounted only update when tokensToHandout != 0
- payoutLastPaid still advances unconditionally
- elapsed time is therefore consumed without ever converting the dust into claimable index value
- the dust can remain stranded in the vault forever
```

## Impact

This issue can cause:

- small reward units to become permanently unclaimable,
- reward accounting to diverge from real vault balances,
- users to observe claim calls that can never recover the stranded inventory,
- and long-running vaults to accumulate irreducible reward dust.

This is an accounting correctness issue rather than a direct theft vector, so Medium severity is appropriate.

## Likelihood

Likelihood is **high**.

The behavior is inherent whenever very small reward balances are introduced and normal accrual continues over time.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because reward balances can become permanently stuck, which breaks exact reward distribution guarantees and leaves vault-held reward inventory that no claimant can ever recover.

### Why it is not High

It is **not high** because the stranded amount is bounded by the dust introduced and does not create an unprivileged path to steal other users’ balances.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. a fresh vault accepts a normal stake,
2. exactly `1` reward token unit is sent to the vault,
3. `poke()` begins the payout timeline,
4. the test then advances by `REWARD_HALF_LIFE * 365`,
5. another `poke()` still leaves `rewardIndex == 0` and `balanceAccounted == 0`,
6. `claimRewards(...)` returns `0`,
7. the reward token balance on the vault remains `1`,
8. proving the dust unit is permanently stranded.

### Test code

### PocPermanentRewardDust.t.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";

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
import { StakingVault } from "@staking/StakingVault.sol";
import { RewardTokenRegistry } from "@staking/RewardTokenRegistry.sol";

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

contract PocPermanentRewardDustTest is Test {
    MockERC20 private underlying;
    MockERC20 private reward;
    StakingVault private stakingVault;

    uint256 private constant REWARD_HALF_LIFE = 3 days;
    uint256 private constant DEPOSIT_AMOUNT = 1_000e18;

    function setUp() public {
        underlying = new MockERC20("Underlying Token", "UNDL");
        reward = new MockERC20("Reward Token", "REWARD");

        MockRoleRegistry roleRegistry = new MockRoleRegistry(address(this));
        ReserveOptimisticGovernanceVersionRegistry versionRegistry =
            new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        RewardTokenRegistry rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

        StakingVault stakingVaultImpl = new StakingVault();
        ReserveOptimisticGovernor governorImpl = new ReserveOptimisticGovernor();
        TimelockControllerOptimistic timelockImpl = new TimelockControllerOptimistic();
        OptimisticSelectorRegistry registryImpl = new OptimisticSelectorRegistry();
        Guardian guardian = new Guardian(address(this), address(0), new address[](0));

        ReserveOptimisticGovernorDeployer deployer = new ReserveOptimisticGovernorDeployer(
            address(versionRegistry),
            address(rewardTokenRegistry),
            address(guardian),
            address(stakingVaultImpl),
            address(governorImpl),
            address(timelockImpl),
            address(registryImpl)
        );

        rewardTokenRegistry.registerRewardToken(address(reward));
        versionRegistry.registerVersion(deployer);

        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = address(reward);

        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams =
            IReserveOptimisticGovernorDeployer.BaseDeploymentParams({
                optimisticParams: IReserveOptimisticGovernor.OptimisticGovernanceParams({
                    vetoDelay: 1 hours, vetoPeriod: 2 hours, vetoThreshold: 0.2e18
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
                proposalThrottleCapacity: 2
            });

        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory newStakingVaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: rewardTokens,
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: 0
            });

        (address stakingVaultAddr,,,) = deployer.deployWithNewStakingVault(baseParams, newStakingVaultParams, bytes32(0));
        stakingVault = StakingVault(stakingVaultAddr);
    }

    function test_poc_singleUnitRewardDustCanNeverReachAnyClaimer() public {
        underlying.mint(address(this), DEPOSIT_AMOUNT);
        underlying.approve(address(stakingVault), DEPOSIT_AMOUNT);
        stakingVault.deposit(DEPOSIT_AMOUNT, address(this));

        reward.mint(address(stakingVault), 1);

        stakingVault.poke();

        vm.warp(block.timestamp + (REWARD_HALF_LIFE * 365));
        stakingVault.poke();

        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = address(reward);
        uint256[] memory claimed = stakingVault.claimRewards(rewardTokens);

        (uint256 payoutLastPaid, uint256 rewardIndex, uint256 balanceAccounted,, uint256 totalClaimed) =
            stakingVault.rewardTrackers(address(reward));

        assertGt(payoutLastPaid, 0, "dust still advances payout windows");
        assertEq(rewardIndex, 0, "dust never increments the global reward index");
        assertEq(balanceAccounted, 0, "dust is never accounted into payouts");
        assertEq(totalClaimed, 0, "no claimer can ever receive the stranded unit");
        assertEq(claimed[0], 0, "claim returns zero even after extreme elapsed time");
        assertEq(reward.balanceOf(address(stakingVault)), 1, "the reward unit remains trapped forever");
    }
}
```

### Test command

```powershell
docker run --rm --entrypoint sh -v "D:\FuckAudit\Governor\Reserve Governor:/work" -w /work ghcr.io/foundry-rs/foundry:latest -lc "forge test --match-path test/poc_new/PocPermanentRewardDust.t.sol -vvv"
```

### Test output

```text
[PASS] test_poc_singleUnitRewardDustCanNeverReachAnyClaimer()
```

