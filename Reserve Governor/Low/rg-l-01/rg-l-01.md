# The Original Optimistic Proposer Can Cancel the Auto-Created Pending Confirmation Proposal

## Metadata

- **Number:** #200
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 2, 2026 at 4:47 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# The Original Optimistic Proposer Can Cancel the Auto-Created Pending Confirmation Proposal
---
**#M-15**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: The same optimistic proposer whose proposal was successfully vetoed can still unilaterally cancel the auto-created confirmation proposal before standard voting begins

## Description

When an optimistic proposal is vetoed, the governor automatically creates a new confirmation proposal in the standard governance lane.

However, that confirmation proposal inherits the original optimistic proposer as its `proposer`, and while it remains `Pending`, the normal proposer-cancel privilege for standard proposals is still available.

That means the same actor whose optimistic proposal was successfully forced into the confirmation lane can simply cancel the new confirmation proposal before the standard voting phase even starts.

This weakens the expected meaning of the veto-transition mechanism: a successful veto can be escalated into the standard lane, but the original optimistic proposer still retains the power to terminate that confirmation proposal during its pending window.

### Confirmation proposal inherits the original proposer

```solidity
function transitionToPessimistic(
    uint256 proposalId,
    IReserveOptimisticGovernor.OptimisticProposalDetails storage optimisticProposal,
    mapping(uint256 proposalId => GovernorUpgradeable.ProposalCore) storage proposalCores
) external {
    ...
    ProposalData memory proposalData = ProposalData(
        newProposalId,
        governor.proposalProposer(proposalId),
        optimisticProposal.targets,
        optimisticProposal.values,
        optimisticProposal.calldatas,
        newDescription
    );

    _saveProposal(proposalData, proposalCores[newProposalId], governor.votingDelay(), governor.votingPeriod());
}
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L109-L143`

The key line is `governor.proposalProposer(proposalId)`: the confirmation proposal is not proposer-neutral; it explicitly reuses the original optimistic proposer.

### Standard proposals are proposer-cancelable while `Pending`

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

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L374-L388`

For non-optimistic proposals — and the confirmation proposal is non-optimistic — the proposer can cancel while the proposal is `Pending`.

### The confirmation proposal starts life in `Pending`

```solidity
proposalCore.voteStart = SafeCast.toUint48(block.timestamp + voteDelay);
proposalCore.voteDuration = SafeCast.toUint32(voteDuration);
```

GitHub:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L183-L185`

So there is always a `Pending` window for the confirmation proposal before standard voting opens.

## Root Cause

The root cause is that the optimistic→confirmation transition reuses the original optimistic proposer while the standard-lane cancellation rule still grants that proposer the normal `Pending`-state cancel privilege.

This allows the original proposer to nullify the very confirmation proposal created to review or override the optimistic path.

## Impact

This issue can cause:

- successful optimistic vetoes to be escalated into the standard lane but then immediately neutralized by the original proposer,
- the confirmation mechanism to be weaker than users may expect,
- and a governance control asymmetry where the original fast-path proposer retains unilateral kill power over the pending confirmation proposal.

This is a governance-process integrity issue rather than direct arbitrary execution or asset theft, so it fits Medium severity.

## Likelihood

Likelihood is **medium**.

The issue requires:

- an optimistic proposer,
- enough `Against` votes to force confirmation,
- and the proposer to react before the confirmation proposal leaves `Pending`.

Those conditions are realistic because the confirmation proposal always begins life in a predictable pending window.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the issue weakens the practical effect of a successful optimistic veto and can suppress a standard-lane review that users might assume is guaranteed once the threshold is reached.

### Why it is not High

It is **not high** because the confirmation proposal still enters the standard lane correctly and the issue does not itself grant arbitrary execution or custody seizure; it is a governance-process control asymmetry.

## Proof of Concept

### What the test proves

The test `test_M15_originalOptimisticProposerCanCancelPendingConfirmationProposal()` demonstrates:

1. an optimistic proposer creates an optimistic proposal,
2. enough `Against` votes force the optimistic proposal into `Defeated`,
3. the governor auto-creates a pending confirmation proposal,
4. and the original optimistic proposer can then cancel that confirmation proposal before standard voting begins.

### Test code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
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

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

contract M15ConfirmationProposerCanCancelPendingConfirmationTest is Test {
    MockERC20 public underlying;
    StakingVault public stakingVault;
    ReserveOptimisticGovernor public governor;

    address public alice = makeAddr("alice");
    address public optimisticProposer = makeAddr("optimisticProposer");

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

    uint256 internal constant ALICE_STAKE = 400_000e18;

    string internal constant CONFIRMATION_PREFIX = "Confirmation For: ";

    function setUp() public {
        underlying = new MockERC20("Underlying Token", "UNDL");

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
        versionRegistry.registerVersion(deployer);

        address[] memory optimisticProposers = new address[](1);
        optimisticProposers[0] = optimisticProposer;

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
                additionalGuardians: new address[](0),
                timelockDelay: TIMELOCK_DELAY,
                proposalThrottleCapacity: PROPOSAL_THROTTLE_CAPACITY
            });

        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory newStakingVaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: new address[](0),
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: 0
            });

        (address stakingVaultAddr, address governorAddr,,) =
            deployer.deployWithNewStakingVault(baseParams, newStakingVaultParams, bytes32(0));

        stakingVault = StakingVault(stakingVaultAddr);
        governor = ReserveOptimisticGovernor(payable(governorAddr));

        underlying.mint(address(this), ALICE_STAKE);
        underlying.approve(address(stakingVault), ALICE_STAKE);
        stakingVault.deposit(ALICE_STAKE, alice);
        vm.prank(alice);
        stakingVault.delegateOptimistic(alice);

        vm.warp(block.timestamp + 12 hours);
    }

    function _warpToActive(uint256 proposalId) internal {
        vm.warp(governor.proposalSnapshot(proposalId) + 1);
    }

    function _confirmationProposalId(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        string memory description
    ) internal view returns (uint256) {
        return governor.getProposalId(targets, values, calldatas, keccak256(bytes(string.concat(CONFIRMATION_PREFIX, description))));
    }

    function test_M15_originalOptimisticProposerCanCancelPendingConfirmationProposal() public {
        address[] memory targets = new address[](1);
        targets[0] = address(underlying);
        uint256[] memory values = new uint256[](1);
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = abi.encodeCall(IERC20.transfer, (alice, 1_000e18));
        string memory description = "confirmation can be proposer-cancelled";

        vm.prank(optimisticProposer);
        uint256 optimisticProposalId = governor.proposeOptimistic(targets, values, calldatas, description);

        _warpToActive(optimisticProposalId);
        vm.prank(alice);
        governor.castVote(optimisticProposalId, 0);

        uint256 confirmationProposalId = _confirmationProposalId(targets, values, calldatas, description);
        assertEq(uint256(governor.state(optimisticProposalId)), uint256(IGovernor.ProposalState.Defeated));
        assertEq(uint256(governor.state(confirmationProposalId)), uint256(IGovernor.ProposalState.Pending));

        vm.prank(optimisticProposer);
        governor.cancel(targets, values, calldatas, keccak256(bytes(string.concat(CONFIRMATION_PREFIX, description))));

        assertEq(uint256(governor.state(confirmationProposalId)), uint256(IGovernor.ProposalState.Canceled));
    }
}
```

### Result

```text
Ran 1 test for test/M15ConfirmationProposerCanCancelPendingConfirmation.t.sol:M15ConfirmationProposerCanCancelPendingConfirmationTest
[PASS] test_M15_originalOptimisticProposerCanCancelPendingConfirmationProposal() (gas: 611102)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

## Recommended Mitigation

- If a successful optimistic veto is intended to guarantee that the matter reaches the standard governance lane, do not give the original optimistic proposer unilateral `Pending`-state cancel rights over the auto-created confirmation proposal.
- Alternatively, make that behavior explicit in the governance model and documentation, so users understand that veto-triggered confirmation can still be terminated by the original proposer before standard voting starts.

