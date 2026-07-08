# maxWithdraw and maxRedeem Ignore Delayed Unstaking and Overstate Immediate Liquidity

## Metadata

- **Number:** #795
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 12:59 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# maxWithdraw and maxRedeem Ignore Delayed Unstaking and Overstate Immediate Liquidity
---
**#M-33**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: ERC4626 integrations can read `maxWithdraw()` and `maxRedeem()` as if the user can receive assets immediately, while the actual vault burns shares and creates a delayed-unstake lock instead of transferring underlying, overstating immediate liquidity and breaking adapter assumptions

## Description

`StakingVault` inherits OpenZeppelin ERC4626 limits unchanged.

That means `maxWithdraw(owner)` and `maxRedeem(owner)` continue to describe the owner’s full share-convertible balance as if a normal ERC4626 withdrawal path were available.

But `StakingVault` no longer behaves like an immediate-settlement ERC4626 vault once `unstakingDelay > 0`.

Under delayed unstaking:

1. `redeem()` still passes the inherited `maxRedeem()` check,
2. the vault burns the user’s shares,
3. but instead of transferring underlying to the receiver, it parks the assets in `UnstakingManager` until unlock,
4. so the user has **zero shares and zero immediate underlying**, even though the ERC4626 limit functions advertised the position as redeemable right now.

This creates a composability mismatch for any adapter or downstream protocol that relies on the ERC4626 limit interface to measure immediately-available liquidity.

The issue is not that delayed unstaking exists.

The issue is that the vault changes withdrawal settlement semantics without also overriding the ERC4626 surface that integrators use to reason about instant redemption capacity.

### OpenZeppelin ERC4626 advertises full withdraw/redeem capacity from share balance alone

```solidity
function maxWithdraw(address owner) public view virtual returns (uint256) {
    return _convertToAssets(balanceOf(owner), Math.Rounding.Floor);
}

function maxRedeem(address owner) public view virtual returns (uint256) {
    return balanceOf(owner);
}
```

Source:
- `node_modules/.pnpm/@openzeppelin+contracts-upgradeable@5.4.0_patch_hash=0bffbc4488ca2b60087b08ff462cd91ca3_5e5089f4ad2b382b638bc43c67641127/node_modules/@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC4626Upgradeable.sol:163-169`

GitHub permalink:
- _local dependency file; no canonical upstream repo permalink generated from this workspace clone_

These functions assume the vault’s standard ERC4626 withdraw/redeem semantics remain truthful.

### `StakingVault` does not override those ERC4626 limit functions

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

function _withdraw(address _caller, address _receiver, address _owner, uint256 _assets, uint256 _shares)
    internal
    override
    accrueRewards(_owner, _receiver)
{
    totalDeposited -= _assets;
    nativeBalanceLastKnown -= _assets;

    if (unstakingDelay == 0) {
        super._withdraw(_caller, _receiver, _owner, _assets, _shares);
    } else {
        if (_caller != _owner) {
            _spendAllowance(_owner, _caller, _shares);
        }

        _burn(_owner, _shares);

        SafeERC20.forceApprove(IERC20(asset()), address(unstakingManager), _assets);
        unstakingManager.createLock(_receiver, _assets, block.timestamp + unstakingDelay);

        emit Withdraw(_caller, _receiver, _owner, _assets, _shares);
    }

    nativeBalanceLastKnown = IERC20(asset()).balanceOf(address(this));
}
```

Source:
- `contracts/staking/StakingVault.sol:240-293`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L240-L293`

The vault customizes `_withdraw()` but leaves `maxWithdraw()` and `maxRedeem()` untouched.

### Delayed unstaking replaces immediate transfer with a time-locked claim path

```solidity
function createLock(address user, uint256 amount, uint256 unlockTime) external {
    require(msg.sender == address(vault), UnstakingManager__Unauthorized());

    SafeERC20.safeTransferFrom(targetToken, msg.sender, address(this), amount);

    uint256 lockId = nextLockId++;
    Lock storage lock = locks[lockId];

    lock.user = user;
    lock.amount = amount;
    lock.unlockTime = unlockTime;

    emit LockCreated(lockId, user, amount, unlockTime);
}

function claimLock(uint256 lockId) external {
    Lock storage lock = locks[lockId];

    require(lock.unlockTime <= block.timestamp && lock.unlockTime != 0, UnstakingManager__NotUnlockedYet());
    require(lock.claimedAt == 0, UnstakingManager__AlreadyClaimed());

    lock.claimedAt = block.timestamp;
    SafeERC20.safeTransfer(targetToken, lock.user, lock.amount);

    emit LockClaimed(lockId);
}
```

Source:
- `contracts/staking/UnstakingManager.sol:40-52`
- `contracts/staking/UnstakingManager.sol:72-80`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/UnstakingManager.sol#L40-L52`
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/UnstakingManager.sol#L72-L80`

So once delayed unstaking is enabled, the user no longer receives assets at redeem time even though ERC4626 limit queries still imply immediate redeemability.

## Root Cause

The root cause is that `StakingVault` changes the concrete settlement semantics of ERC4626 withdrawals when `unstakingDelay > 0`, but it does not override the inherited ERC4626 limit interface to reflect that assets are no longer immediately withdrawable.

```text
root cause:
- OpenZeppelin ERC4626 maxWithdraw/maxRedeem describe withdrawal capacity from share balance only
- StakingVault inherits those functions unchanged
- StakingVault overrides _withdraw() so delayed unstaking burns shares and creates a lock instead of transferring underlying immediately
- the ERC4626 limit surface therefore overstates immediate liquidity once unstakingDelay is enabled
```

## Impact

This issue can cause:

- ERC4626 adapters to overestimate immediately withdrawable assets,
- liquidity managers and aggregators to assume instant redemption where only delayed claim rights exist,
- users to lose shares immediately while receiving no immediate underlying,
- and downstream integrations to mis-handle slippage, solvency, or exit-routing decisions because the vault advertises a stronger immediate liquidity guarantee than it actually provides.

This is a composability and interface-integrity issue rather than direct theft, so Medium severity is appropriate.

## Likelihood

Likelihood is **medium**.

The issue requires:

- delayed unstaking to be enabled,
- and an integration or workflow that relies on ERC4626 `maxWithdraw()` / `maxRedeem()` as an immediate-liquidity oracle.

That is a common enough integration assumption in ERC4626 ecosystems that the mismatch is realistic, even though it is not triggered in every deployment by default.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the vault exposes misleading ERC4626 liquidity limits that can break downstream assumptions and produce incorrect redemption behavior for users and adapters.

### Why it is not High

It is **not high** because the issue does not directly let an attacker steal funds or bypass authorization. The core risk is inaccurate interface truth and resulting composability failure.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. delayed unstaking is enabled,
2. `maxWithdraw(alice)` reports Alice’s full stake as withdrawable,
3. `maxRedeem(alice)` reports Alice’s full share balance as redeemable,
4. Alice calls `redeem()` for that full amount,
5. her shares are burned immediately,
6. but her underlying token balance does not increase at all,
7. and the vault instead creates a delayed `UnstakingManager` lock for the full amount.

### Test code

### PocNewBase.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

import { OptimisticSelectorRegistry } from "@governance/OptimisticSelectorRegistry.sol";
import { ReserveOptimisticGovernor } from "@governance/ReserveOptimisticGovernor.sol";
import { TimelockControllerOptimistic } from "@governance/TimelockControllerOptimistic.sol";
import { IReserveOptimisticGovernorDeployer } from "@interfaces/IDeployer.sol";
import { IOptimisticSelectorRegistry } from "@interfaces/IOptimisticSelectorRegistry.sol";
import { IReserveOptimisticGovernor } from "@interfaces/IReserveOptimisticGovernor.sol";
import { ReserveOptimisticGovernorDeployer } from "@src/Deployer.sol";
import { Guardian } from "@src/Guardian.sol";
import { ReserveOptimisticGovernanceVersionRegistry } from "@src/VersionRegistry.sol";
import { RewardTokenRegistry } from "@staking/RewardTokenRegistry.sol";
import { StakingVault } from "@staking/StakingVault.sol";
import { CANCELLER_ROLE, EXECUTOR_ROLE, OPTIMISTIC_PROPOSER_ROLE, PROPOSER_ROLE } from "@utils/Constants.sol";

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

abstract contract PocNewBase is Test {
    MockERC20 internal underlying;
    StakingVault internal stakingVault;
    OptimisticSelectorRegistry internal registry;
    Guardian internal guardianContract;
    ReserveOptimisticGovernorDeployer internal deployer;
    ReserveOptimisticGovernor internal governor;
    TimelockControllerOptimistic internal timelock;

    address internal alice = makeAddr("alice");
    address internal bob = makeAddr("bob");
    address internal carol = makeAddr("carol");
    address internal guardian = makeAddr("guardian");
    address internal additionalGuardian = makeAddr("additionalGuardian");
    address internal optimisticGuardianManager = makeAddr("optimisticGuardianManager");
    address internal optimisticGuardian = makeAddr("optimisticGuardian");
    address internal optimisticProposer = makeAddr("optimisticProposer");
    address internal optimisticProposer2 = makeAddr("optimisticProposer2");

    uint48 internal constant VETO_DELAY = 1 hours;
    uint32 internal constant VETO_PERIOD = 2 hours;
    uint256 internal constant VETO_THRESHOLD = 0.2e18;
    uint48 internal constant VOTING_DELAY = 1 days;
    uint32 internal constant VOTING_PERIOD = 1 weeks;
    uint48 internal constant VOTE_EXTENSION = 1 days;
    uint256 internal constant PROPOSAL_THRESHOLD = 0.01e18;
    uint256 internal constant QUORUM_NUMERATOR = 0.1e18;
    uint256 internal constant PROPOSAL_THROTTLE_CAPACITY = 2;
    uint256 internal constant TIMELOCK_DELAY = 2 days;
    uint256 internal constant REWARD_HALF_LIFE = 1 days;
    uint256 internal constant UNSTAKING_DELAY = 0;
    uint256 internal constant ALICE_STAKE = 400_000e18;
    uint256 internal constant BOB_STAKE = 400_000e18;
    uint256 internal constant CAROL_STAKE = 200_000e18;

    function _useExistingStakingVaultDeployment() internal pure virtual returns (bool);

    function setUp() public virtual {
        underlying = new MockERC20("Underlying Token", "UNDL");

        MockRoleRegistry roleRegistry = new MockRoleRegistry(address(this));
        ReserveOptimisticGovernanceVersionRegistry versionRegistry =
            new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        RewardTokenRegistry rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

        StakingVault stakingVaultImpl = new StakingVault();
        ReserveOptimisticGovernor governorImpl = new ReserveOptimisticGovernor();
        TimelockControllerOptimistic timelockImpl = new TimelockControllerOptimistic();
        OptimisticSelectorRegistry registryImpl = new OptimisticSelectorRegistry();
        address[] memory optimisticGuardians = new address[](1);
        optimisticGuardians[0] = optimisticGuardian;

        guardianContract = new Guardian(guardian, optimisticGuardianManager, optimisticGuardians);

        deployer = new ReserveOptimisticGovernorDeployer(
            address(versionRegistry),
            address(rewardTokenRegistry),
            address(guardianContract),
            address(stakingVaultImpl),
            address(governorImpl),
            address(timelockImpl),
            address(registryImpl)
        );
        versionRegistry.registerVersion(deployer);

        address[] memory optimisticProposers = new address[](2);
        optimisticProposers[0] = optimisticProposer;
        optimisticProposers[1] = optimisticProposer2;

        bytes4[] memory transferSelectors = new bytes4[](1);
        transferSelectors[0] = IERC20.transfer.selector;

        IOptimisticSelectorRegistry.SelectorData[] memory selectorData =
            new IOptimisticSelectorRegistry.SelectorData[](1);
        selectorData[0] = IOptimisticSelectorRegistry.SelectorData(address(underlying), transferSelectors);

        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams =
            IReserveOptimisticGovernorDeployer.BaseDeploymentParams({
                optimisticParams: IReserveOptimisticGovernor.OptimisticGovernanceParams({
                    vetoDelay: VETO_DELAY, vetoPeriod: VETO_PERIOD, vetoThreshold: VETO_THRESHOLD
                }),
                standardParams: IReserveOptimisticGovernor.StandardGovernanceParams({
                    votingDelay: VOTING_DELAY,
                    votingPeriod: VOTING_PERIOD,
                    voteExtension: VOTE_EXTENSION,
                    proposalThreshold: PROPOSAL_THRESHOLD,
                    quorumNumerator: QUORUM_NUMERATOR
                }),
                selectorData: selectorData,
                optimisticProposers: optimisticProposers,
                additionalGuardians: _additionalGuardians(),
                timelockDelay: TIMELOCK_DELAY,
                proposalThrottleCapacity: PROPOSAL_THROTTLE_CAPACITY
            });

        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory newStakingVaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: new address[](0),
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: UNSTAKING_DELAY
            });

        (address stakingVaultAddr, address governorAddr, address timelockAddr, address selectorRegistryAddr) =
            deployer.deployWithNewStakingVault(baseParams, newStakingVaultParams, bytes32(0));

        if (_useExistingStakingVaultDeployment()) {
            (address existingGovernorAddr, address existingTimelockAddr, address existingSelectorRegistryAddr) =
                deployer.deployWithExistingStakingVault(baseParams, stakingVaultAddr, bytes32(uint256(1)));

            governor = ReserveOptimisticGovernor(payable(existingGovernorAddr));
            timelock = TimelockControllerOptimistic(payable(existingTimelockAddr));
            registry = OptimisticSelectorRegistry(existingSelectorRegistryAddr);
            stakingVault = StakingVault(address(governor.token()));
        } else {
            governor = ReserveOptimisticGovernor(payable(governorAddr));
            timelock = TimelockControllerOptimistic(payable(timelockAddr));
            registry = OptimisticSelectorRegistry(selectorRegistryAddr);
            stakingVault = StakingVault(stakingVaultAddr);
        }

        _setupVoter(alice, ALICE_STAKE);
        _setupVoter(bob, BOB_STAKE);
        _setupVoter(carol, CAROL_STAKE);

        vm.warp(block.timestamp + 12 hours);
    }

    function _additionalGuardians() internal view returns (address[] memory guardians) {
        guardians = new address[](1);
        guardians[0] = additionalGuardian;
    }

    function _setupVoter(address voter, uint256 amount) internal {
        underlying.mint(voter, amount);

        vm.startPrank(voter);
        underlying.approve(address(stakingVault), amount);
        stakingVault.depositAndDelegate(amount);
        vm.stopPrank();
    }

    function _singleCall(address target, uint256 value, bytes memory callData)
        internal
        pure
        returns (address[] memory targets, uint256[] memory values, bytes[] memory calldatas)
    {
        targets = new address[](1);
        values = new uint256[](1);
        calldatas = new bytes[](1);

        targets[0] = target;
        values[0] = value;
        calldatas[0] = callData;
    }

    function _warpToActive(uint256 proposalId) internal {
        vm.warp(governor.proposalSnapshot(proposalId) + 1);
    }

    function _warpPastDeadline(uint256 proposalId) internal {
        vm.warp(governor.proposalDeadline(proposalId) + 1);
    }

    function _proposePassAndQueueStandard(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) internal returns (uint256 proposalId, bytes32 descriptionHash) {
        vm.prank(alice);
        proposalId = governor.propose(targets, values, calldatas, description);

        _warpToActive(proposalId);

        vm.prank(alice);
        governor.castVote(proposalId, 1);
        vm.prank(bob);
        governor.castVote(proposalId, 1);
        vm.prank(carol);
        governor.castVote(proposalId, 1);

        _warpPastDeadline(proposalId);
        descriptionHash = keccak256(bytes(description));
        governor.queue(targets, values, calldatas, descriptionHash);
    }
}

abstract contract PocNewBaseNewVault is PocNewBase {
    function _useExistingStakingVaultDeployment() internal pure virtual override returns (bool) {
        return false;
    }
}

abstract contract PocNewBaseExistingVault is PocNewBase {
    function _useExistingStakingVaultDeployment() internal pure virtual override returns (bool) {
        return true;
    }
}
```

### maxWithdraw and maxRedeem Ignore Delayed Unstaking and Overstate Immediate Liquidity

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { UnstakingManager } from "@staking/UnstakingManager.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocLockUnawareMaxWithdrawTest is PocNewBaseNewVault {
    function test_poc_maxWithdrawAndRedeemIgnoreUnstakingDelayImmediateLiquidity() public {
        vm.prank(address(timelock));
        stakingVault.setUnstakingDelay(14 days);

        uint256 reportedMaxWithdraw = stakingVault.maxWithdraw(alice);
        uint256 reportedMaxRedeem = stakingVault.maxRedeem(alice);

        assertEq(reportedMaxWithdraw, ALICE_STAKE, "maxWithdraw reports full stake as withdrawable");
        assertEq(reportedMaxRedeem, ALICE_STAKE, "maxRedeem reports full stake as redeemable");

        uint256 aliceUnderlyingBefore = underlying.balanceOf(alice);

        vm.prank(alice);
        stakingVault.redeem(ALICE_STAKE, alice, alice);

        assertEq(
            underlying.balanceOf(alice),
            aliceUnderlyingBefore,
            "redeem does not transfer assets immediately when unstaking delay is enabled"
        );
        assertEq(stakingVault.balanceOf(alice), 0, "shares are burned despite maxRedeem advertising immediate redeemability");

        UnstakingManager manager = stakingVault.unstakingManager();
        (address user, uint256 amount, uint256 unlockTime, uint256 claimedAt) = manager.locks(0);
        assertEq(user, alice, "lock should belong to redeemer");
        assertEq(amount, ALICE_STAKE, "full redeem amount should be parked in lock");
        assertEq(claimedAt, 0, "lock starts unclaimed");
        assertGt(unlockTime, block.timestamp, "unlock remains delayed after redeem");
    }
}
```

### Test output

```text
Ran 1 test for test/poc_new/PocLockUnawareMaxWithdraw.t.sol:PocLockUnawareMaxWithdrawTest
[PASS] test_poc_maxWithdrawAndRedeemIgnoreUnstakingDelayImmediateLiquidity() (gas: 345401)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 25.62ms (710.99µs CPU time)

Ran 1 test suite in 47.24ms (25.62ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendation

- Override `maxWithdraw()` and `maxRedeem()` so they reflect the delayed-unstaking reality rather than immediate settlement capacity.
- If delayed settlement is intentional and should remain ERC4626-compatible only at the conversion layer, consider returning `0` for immediate-withdraw limit functions while `unstakingDelay > 0`, or otherwise expose a separate delayed-claim interface that integrators can reason about safely.
- More generally, make sure the ERC4626 surface describes not just mathematical convertibility but the actual settlement guarantees that a withdraw/redeem call provides.

