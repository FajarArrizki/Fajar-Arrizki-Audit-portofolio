# Undelegated Deposits Inflate Quorum Without Adding Votes

## Metadata

- **Number:** #844
- **Severity:** Medium
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 6:45 AM
- **Last updated:** June 2, 2026 at 5:52 AM
- **Reward:** 0

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Undelegated Deposits Inflate Quorum Without Adding Votes
---
**#M-49**
- Severity: Medium
- Validity: Tested
- Likelihood: High
- Impact: A large passive staker can raise quorum by increasing total share supply while contributing zero delegated voting power, which can push quorum above the entire delegated electorate and make otherwise unanimous governance proposals fail.

## Description

The staking vault separates deposit from delegation.

Plain deposits mint shares immediately, but voting power only becomes usable after the holder delegates through the standard and optimistic delegation paths.

That would be fine if governance quorum were based on delegated votes.

It is not.

`ReserveOptimisticGovernor.quorum(...)` ultimately derives quorum from `token().getPastTotalSupply(...)`, which counts all minted vault shares, including undelegated supply.

So a passive depositor can meaningfully increase the quorum denominator without increasing the votes that any account can actually cast.

The result is a liveness break: governance can become impossible even when every delegated holder votes in favor.

### Deposits mint shares immediately, while delegation is an explicit separate step

```solidity
function depositAndDelegate(uint256 assets, address delegatee, address optimisticDelegatee)
    public
    returns (uint256 shares)
{
    shares = deposit(assets, msg.sender);

    _delegate(msg.sender, delegatee);
    _delegateOptimistic(msg.sender, optimisticDelegatee);
}

function getOptimisticVotes(address account) external view returns (uint256) {
    return optimisticDelegateCheckpoints[account].latest();
}
```

Source:
- `contracts/staking/StakingVault.sol:188-196`
- `contracts/staking/StakingVault.sol:220-221`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L188-L196`
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L220-L221`

This shows delegation is opt-in behavior layered on top of deposit.

### Quorum is computed from past total supply, not past delegated voting supply

```solidity
function quorum(uint256 timepoint)
    public
    view
    override(GovernorUpgradeable, GovernorVotesQuorumFractionUpgradeable)
    returns (uint256)
{
    return Math.max(1, super.quorum(timepoint));
}

function proposalThreshold()
    public
    view
    override(GovernorUpgradeable, GovernorSettingsUpgradeable)
    returns (uint256)
{
    uint256 proposalThresholdRatio = super.proposalThreshold(); // D18{1}

    // {tok}
    uint256 supply = Math.max(1, token().getPastTotalSupply(block.timestamp - 1));
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:202-208`
- `contracts/governance/ReserveOptimisticGovernor.sol:302-314`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L202-L208`
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L302-L314`

Because undelegated shares still expand total share supply, quorum grows even though those shares add no active voting power.

## Root Cause

The root cause is a mismatch between the unit used for governance participation and the unit used for governance quorum.

```text
root cause:
- staking deposits mint ERC4626 shares immediately
- delegation is optional and separate from deposit
- undelegated shares therefore add zero usable votes
- governor quorum still keys off past total share supply
- passive supply can raise quorum without adding any castable voting power
```

## Impact

This issue can cause:

- passive capital to dilute governance liveness,
- unanimous delegated voting blocs to fail quorum anyway,
- quorum to become meaningfully detached from the set of accounts that can actually vote,
- and governance operators to misread a healthy total supply increase as healthy governance participation.

This is a governance liveness and accounting-integrity issue rather than direct fund theft, so Medium severity is appropriate.

## Likelihood

Likelihood is **high**.

The behavior is inherent to the current design whenever deposits are made without delegation, which is a normal and permissionless user path.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the bug can make valid governance proposals fail even when the full delegated electorate supports them. That breaks core governance functionality.

### Why it is not High

It is **not high** because the issue does not directly transfer value, seize roles, or allow arbitrary execution. It degrades governance liveness and quorum correctness.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. a passive staker deposits `20_000_000e18` underlying,
2. the staker receives the full share balance,
3. the staker still has `0` standard votes and `0` optimistic votes because they never delegated,
4. governance quorum nevertheless increases because it counts past total supply,
5. Alice, Bob, and Carol cast all delegated votes in favor,
6. the proposal still ends `Defeated` because undelegated supply pushed quorum above the delegated electorate.

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

### PocUndelegatedDepositQuorumInflation.t.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocUndelegatedDepositQuorumInflationTest is PocNewBaseNewVault {
    function test_poc_undelegatedDepositInflatesQuorumWithoutAddingVotes() public {
        address passiveStaker = makeAddr("passiveStaker");
        uint256 passiveDeposit = 20_000_000e18;

        underlying.mint(passiveStaker, passiveDeposit);

        vm.startPrank(passiveStaker);
        underlying.approve(address(stakingVault), passiveDeposit);
        stakingVault.deposit(passiveDeposit, passiveStaker);
        vm.stopPrank();

        assertEq(stakingVault.balanceOf(passiveStaker), passiveDeposit, "passive staker still receives full shares");
        assertEq(stakingVault.getVotes(passiveStaker), 0, "undelegated shares add no standard voting power");
        assertEq(stakingVault.getOptimisticVotes(passiveStaker), 0, "undelegated shares add no optimistic voting power");

        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
            _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (alice, 1e18)));

        vm.prank(alice);
        uint256 proposalId = governor.propose(targets, values, calldatas, "undelegated quorum inflation");

        uint256 snapshot = governor.proposalSnapshot(proposalId);
        vm.warp(snapshot + 1);
        uint256 inflatedQuorum = governor.quorum(snapshot);
        uint256 totalDelegatedVotes = ALICE_STAKE + BOB_STAKE + CAROL_STAKE;

        assertGt(inflatedQuorum, totalDelegatedVotes, "undelegated supply pushes quorum above all delegated votes");

        _warpToActive(proposalId);

        vm.prank(alice);
        governor.castVote(proposalId, 1);
        vm.prank(bob);
        governor.castVote(proposalId, 1);
        vm.prank(carol);
        governor.castVote(proposalId, 1);

        (, uint256 forVotes,) = governor.proposalVotes(proposalId);
        assertEq(forVotes, totalDelegatedVotes, "all delegated stake votes in favor");

        _warpPastDeadline(proposalId);

        assertEq(
            uint256(governor.state(proposalId)),
            uint256(IGovernor.ProposalState.Defeated),
            "proposal still fails because quorum counts undelegated supply"
        );
    }
}
```

### Test command

```powershell
docker run --rm --entrypoint sh -v "D:\FuckAudit\Governor\Reserve Governor:/work" -w /work ghcr.io/foundry-rs/foundry:latest -lc "forge test --match-path test/poc_new/PocUndelegatedDepositQuorumInflation.t.sol -vvv"
```

### Test output

```text
[PASS] test_poc_undelegatedDepositInflatesQuorumWithoutAddingVotes()
```

