# Unbounded Optimistic Proposal Descriptions Persist and Duplicate Across Confirmation Transition

## Metadata

- **Number:** #855
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 6:59 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Unbounded Optimistic Proposal Descriptions Persist and Duplicate Across Confirmation Transition
---
**#M-58**
- Severity: Medium
- Validity: Tested
- Likelihood: High
- Impact: Optimistic proposals accept arbitrarily large descriptions, store them onchain, emit them in `ProposalCreated`, and if vetoed duplicate the entire payload again into the confirmation proposal, creating avoidable storage and event-gas bloat in a path that the shared guardian may need to react to under stress.

## Description

Optimistic proposal admission validates description shape, but it never enforces any size limit.

When an optimistic proposal is created:

1. the full `string description` is stored inside `optimisticProposalDetails`,
2. the same description is emitted through `ProposalCreated`,
3. and if the proposal is vetoed into the standard lane, `transitionToPessimistic()` builds `Confirmation For: ` plus the entire original description and saves that as a second proposal.

So a single oversized optimistic description becomes duplicated governance state.

This is not a correctness bug in proposal hashing. The issue is that the protocol offers no bound on a field that is persisted, re-emitted, and copied into a second proposal lifecycle.

### `proposeOptimistic()` stores the full caller-supplied description in optimistic proposal state

```solidity
function proposeOptimistic(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata calldatas,
    string calldata description
) external returns (uint256 proposalId) {
    proposalId = getProposalId(targets, values, calldatas, keccak256(bytes(description)));

    optimisticProposalDetails[proposalId] = OptimisticProposalDetails({
        targets: targets,
        values: values,
        calldatas: calldatas,
        description: description,
        vetoThreshold: optimisticParams.vetoThreshold
    });

    ProposalLib.proposeOptimistic(
        ProposalLib.ProposalData(proposalId, proposer, targets, values, calldatas, description),
        _proposalCore(proposalId),
        optimisticParams
    );
}
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:149-173`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L149-L173`

### Proposal validation checks format, not size, and `_saveProposal()` emits the full description

```solidity
function _validateProposal(ProposalData calldata proposal, GovernorUpgradeable.ProposalCore storage proposalCore)
    private
    view
{
    require(
        _isValidDescriptionForProposer(proposal.proposer, proposal.description),
        IGovernor.GovernorRestrictedProposer(proposal.proposer)
    );

    require(
        bytes18(bytes(proposal.description)) != CONFIRMATION_PREFIX_BYTES,
        IReserveOptimisticGovernor.OptimisticGovernor__ConfirmationPrefixNotAllowed()
    );
}

function _saveProposal(
    ProposalData memory proposal,
    GovernorUpgradeable.ProposalCore storage proposalCore,
    uint256 voteDelay,
    uint256 voteDuration
) private {
    emit IGovernor.ProposalCreated(
        proposal.proposalId,
        proposal.proposer,
        proposal.targets,
        proposal.values,
        new string[](proposal.targets.length),
        proposal.calldatas,
        proposalCore.voteStart,
        proposalCore.voteStart + proposalCore.voteDuration,
        proposal.description
    );
}
```

Source:
- `contracts/governance/lib/ProposalLib.sol:147-175`
- `contracts/governance/lib/ProposalLib.sol:177-198`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L147-L175`
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L177-L198`

### Veto transition duplicates the oversized description into a second proposal

```solidity
function transitionToPessimistic(
    uint256 proposalId,
    IReserveOptimisticGovernor.OptimisticProposalDetails storage optimisticProposal,
    mapping(uint256 proposalId => GovernorUpgradeable.ProposalCore) storage proposalCores
) external {
    string memory newDescription = string.concat(CONFIRMATION_PREFIX, optimisticProposal.description);

    uint256 newProposalId = governor.getProposalId(
        optimisticProposal.targets,
        optimisticProposal.values,
        optimisticProposal.calldatas,
        keccak256(bytes(newDescription))
    );

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

Source:
- `contracts/governance/lib/ProposalLib.sol:108-143`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L108-L143`

So a single huge description can become two persisted proposal descriptions plus two event payloads.

## Root Cause

The root cause is that description text is treated as unbounded governance metadata even though it is persisted in proposal state and copied again during optimistic-to-confirmation transition.

```text
root cause:
- optimistic proposal creation accepts arbitrary-length descriptions
- ReserveOptimisticGovernor persists the full string inside optimisticProposalDetails
- ProposalLib emits the full string in ProposalCreated
- transitionToPessimistic() prepends a prefix and saves the whole payload again as a second proposal
- no upper bound exists to cap the storage and event footprint of this duplicated metadata
```

## Impact

This issue can cause:

- avoidable proposal-creation gas bloat,
- duplicated storage and event payloads when optimistic proposals are vetoed,
- more expensive governance reactions in stressed scenarios,
- and a griefable metadata surface for allowed optimistic proposers.

This is a state-growth and governance-liveness issue rather than a direct execution bypass, so Medium severity is appropriate.

## Likelihood

Likelihood is **high**.

Any address with `OPTIMISTIC_PROPOSER_ROLE` can supply an oversized description immediately, and the duplication path is automatic if the proposal reaches the confirmation lane.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the protocol stores and duplicates unbounded governance metadata in a way that can raise costs and operational friction during proposal handling.

### Why it is not High

It is **not high** because the bug does not directly alter proposal authorization, vote counting, or fund movement. It bloats storage and execution costs.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. an optimistic proposal is created with a 16,384-byte description,
2. the proposal is accepted despite that unbounded payload,
3. the proposal is vetoed into the confirmation lane,
4. the governor derives the confirmation proposal from `Confirmation For: ` plus the entire huge description,
5. the confirmation proposal is created successfully,
6. proving oversized descriptions persist and are duplicated across the transition.

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

### PocUnboundedOptimisticDescriptionStorageBloat.t.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocUnboundedOptimisticDescriptionStorageBloatTest is PocNewBaseNewVault {
    function test_poc_optimisticProposalAcceptsHugeDescriptionAndDefeatDuplicatesItIntoConfirmationLane() public {
        string memory hugeDescription = _buildLargeDescription(16_384);

        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
            _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (alice, 1e18)));

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(targets, values, calldatas, hugeDescription);

        _warpToActive(proposalId);

        vm.prank(bob);
        governor.castVote(proposalId, 0);

        string memory confirmationDescription = string.concat("Confirmation For: ", hugeDescription);
        uint256 confirmationProposalId =
            governor.hashProposal(targets, values, calldatas, keccak256(bytes(confirmationDescription)));

        assertEq(
            uint256(governor.state(proposalId)),
            uint256(IGovernor.ProposalState.Defeated),
            "optimistic proposal can be defeated even when carrying a huge unbounded description"
        );
        assertEq(
            uint256(governor.state(confirmationProposalId)),
            uint256(IGovernor.ProposalState.Pending),
            "defeat copies the oversized description into a second confirmation proposal"
        );
        assertEq(
            governor.proposalSnapshot(confirmationProposalId),
            block.timestamp + VOTING_DELAY,
            "confirmation proposal is created from the duplicated oversized description"
        );
    }

    function _buildLargeDescription(uint256 length) internal pure returns (string memory) {
        bytes memory payload = new bytes(length);

        for (uint256 i = 0; i < length; ++i) {
            payload[i] = bytes1(uint8(65 + (i % 26)));
        }

        return string(payload);
    }
}
```

### Test command

```powershell
docker run --rm --entrypoint sh -v "D:\FuckAudit\Governor\Reserve Governor:/work" -w /work ghcr.io/foundry-rs/foundry:latest -lc "forge test --match-path test/poc_new/PocUnboundedOptimisticDescriptionStorageBloat.t.sol -vvv"
```

### Test output

```text
[PASS] test_poc_optimisticProposalAcceptsHugeDescriptionAndDefeatDuplicatesItIntoConfirmationLane()
```

