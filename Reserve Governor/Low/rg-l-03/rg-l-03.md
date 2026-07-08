# Optimistic Veto Threshold Rounds Down Instead of Up

## Metadata

- **Number:** #752
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 8, 2026 at 9:11 PM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Optimistic Veto Threshold Rounds Down Instead of Up
---
**#M-23**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: Fractional optimistic veto thresholds are rounded down, letting fewer AGAINST votes than documented defeat fast proposals and force confirmation votes early

## Description

The README documents optimistic veto threshold calculation as:

```text
vetoThreshold = ceil(vetoThresholdRatio * pastTotalSupply / 1e18)
```

However, the actual optimistic proposal state logic computes the threshold in token units using floor division:

```solidity
uint256 vetoThresholdTok = (_vetoThreshold * pastSupply) / 1e18;
vetoThresholdTok = Math.max(vetoThresholdTok, 1);
```

That difference matters whenever `vetoThresholdRatio * pastSupply / 1e18` is fractional.

For example, with:

- `vetoThresholdRatio = 10%`
- `pastSupply = 11`

the documented threshold is:

- `ceil(1.1) = 2`

but the implemented threshold is:

- `floor(1.1) = 1`

So a single `Against` vote can already defeat the optimistic proposal and spawn a confirmation proposal even though the documented threshold would still require two votes.

This makes the optimistic lane easier to veto than governance users, documentation, and integrators are told to expect.

### Implementation floors the optimistic veto threshold

```solidity
// {tok} = D18{1} * {tok} / D18{1}
uint256 vetoThresholdTok = (_vetoThreshold * pastSupply) / 1e18;
vetoThresholdTok = Math.max(vetoThresholdTok, 1);

// {tok}
(uint256 againstVotes,,) = proposalVotes(proposalId);

if (againstVotes >= vetoThresholdTok) {
    return ProposalState.Defeated;
}
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:255-263`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L255-L263`

### The same contract already uses ceil rounding for proposal thresholds

```solidity
// CEIL to make sure thresholds near 0% don't get rounded down to 0 tokens
return (proposalThresholdRatio * supply + (1e18 - 1)) / 1e18;
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:313-314`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L313-L314`

So the optimistic veto path is internally inconsistent with the contract’s own threshold-rounding approach elsewhere.

### The README documents ceil semantics explicitly

```text
vetoThreshold = ceil(vetoThresholdRatio * pastTotalSupply / 1e18)
```

Source:
- `README.md:164-172`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/README.md#L164-L172`

## Root Cause

The root cause is that optimistic veto threshold conversion from ratio-space into token-space uses floor division instead of ceil division, even though the documented semantics and related threshold logic expect rounding up.

```text
root cause:
- optimistic veto thresholds are stored as D18 ratios
- proposal state converts that ratio into token units with floor division
- fractional thresholds therefore resolve to the next lower whole-token count
- the fast-path defeat/confirmation transition can trigger one vote earlier than documented
```

## Impact

This issue can cause:

- optimistic proposals to be defeated with fewer `Against` votes than governance expects,
- confirmation votes to be spawned earlier than the documented threshold,
- narrower optimistic execution reliability around low-supply or low-ratio configurations,
- and governance parameterization mistakes if operators assume the README’s ceil semantics are enforced onchain.

This is primarily a governance-threshold integrity issue rather than a direct theft primitive, so Medium severity is appropriate.

## Likelihood

Likelihood is **medium**.

The bug is deterministic whenever:

- the optimistic veto ratio multiplied by snapshot supply is fractional, and
- that fractional part would have required an extra whole vote under ceil semantics.

That will not happen on every configuration, but when it does happen the discrepancy is guaranteed and reproducible.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because it changes the actual governance threshold for defeating fast proposals, making the optimistic path easier to block than documented and causing earlier-than-expected forced confirmation votes.

### Why it is not High

It is **not high** because the issue does not by itself grant arbitrary execution, seize funds, or bypass access control. It weakens threshold integrity and governance predictability rather than creating direct asset theft.

## Proof of Concept

### What the tests prove

The standalone PoC demonstrates:

1. a tiny optimistic voting supply of `11` tokens is created,
2. the optimistic veto ratio is configured as `10%`,
3. the documented threshold is therefore `ceil(1.1) = 2`,
4. the implementation instead computes `floor(1.1) = 1`,
5. a single `Against` vote defeats the optimistic proposal,
6. and that one vote also spawns the confirmation proposal immediately.

### Test code

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

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

contract PocOptimisticVetoThresholdRoundsDownTest is Test {
    MockERC20 internal underlying;
    StakingVault internal stakingVault;
    ReserveOptimisticGovernor internal governor;

    address internal alice = makeAddr("alice");
    address internal bob = makeAddr("bob");
    address internal optimisticProposer = makeAddr("optimisticProposer");

    uint48 internal constant VETO_DELAY = 1 hours;
    uint32 internal constant VETO_PERIOD = 2 hours;
    uint256 internal constant VETO_THRESHOLD = 0.1e18; // 10%
    uint48 internal constant VOTING_DELAY = 1 days;
    uint32 internal constant VOTING_PERIOD = 1 weeks;
    uint48 internal constant VOTE_EXTENSION = 1 days;
    uint256 internal constant PROPOSAL_THRESHOLD = 0.01e18;
    uint256 internal constant QUORUM_NUMERATOR = 0.1e18;
    uint256 internal constant PROPOSAL_THROTTLE_CAPACITY = 2;
    uint256 internal constant TIMELOCK_DELAY = 2 days;
    uint256 internal constant REWARD_HALF_LIFE = 1 days;

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
                    vetoDelay: VETO_DELAY,
                    vetoPeriod: VETO_PERIOD,
                    vetoThreshold: VETO_THRESHOLD
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

        _setupVoter(alice, 10);
        _setupVoter(bob, 1);

        vm.warp(block.timestamp + 12 hours);
    }

    function test_fractionalVetoThresholdRoundsDownAndDefeatsTooEarly() public {
        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
            _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (address(0xBEEF), 1)));

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(targets, values, calldatas, "fractional-veto-threshold");

        _warpToActive(proposalId);

        uint256 snapshot = governor.proposalSnapshot(proposalId);
        uint256 pastSupply = stakingVault.getPastTotalSupply(snapshot);
        assertEq(pastSupply, 11, "expected tiny total supply for rounding edge");

        uint256 documentedCeilThreshold = (VETO_THRESHOLD * pastSupply + (1e18 - 1)) / 1e18;
        uint256 implementedFloorThreshold = (VETO_THRESHOLD * pastSupply) / 1e18;

        assertEq(documentedCeilThreshold, 2, "README ceil threshold should require 2 wei votes");
        assertEq(implementedFloorThreshold, 1, "implementation floors to 1 wei vote");

        vm.prank(bob);
        governor.castVote(proposalId, 0);

        (uint256 againstVotes,,) = governor.proposalVotes(proposalId);
        assertEq(againstVotes, 1, "single wei vote should be counted");
        assertEq(uint256(governor.state(proposalId)), uint256(IGovernor.ProposalState.Defeated));
    }

    function test_singleAgainstVoteSpawnsConfirmationProposalBelowDocumentedCeilThreshold() public {
        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
            _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (address(0xBEEF), 1)));
        string memory description = "fractional-veto-threshold-confirmation";

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(targets, values, calldatas, description);

        _warpToActive(proposalId);

        vm.prank(bob);
        governor.castVote(proposalId, 0);

        uint256 confirmationProposalId = governor.getProposalId(
            targets,
            values,
            calldatas,
            keccak256(bytes(string.concat("Confirmation For: ", description)))
        );

        assertEq(uint256(governor.state(proposalId)), uint256(IGovernor.ProposalState.Defeated));
        assertEq(uint256(governor.state(confirmationProposalId)), uint256(IGovernor.ProposalState.Pending));
    }

    function _setupVoter(address voter, uint256 amount) internal {
        underlying.mint(address(this), amount);
        underlying.approve(address(stakingVault), amount);
        stakingVault.deposit(amount, voter);

        vm.prank(voter);
        stakingVault.delegateOptimistic(voter);
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
}
```

### Test output

```text
Compiling 1 files with Solc 0.8.33
Solc 0.8.33 finished in 7.49s
Compiler run successful!

Ran 2 tests for test/poc_new/PocOptimisticVetoThresholdRoundsDown.t.sol:PocOptimisticVetoThresholdRoundsDownTest
[PASS] test_fractionalVetoThresholdRoundsDownAndDefeatsTooEarly() (gas: 538141)
[PASS] test_singleAgainstVoteSpawnsConfirmationProposalBelowDocumentedCeilThreshold() (gas: 586471)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 32.66ms (3.10ms CPU time)

Ran 1 test suite in 61.08ms (32.66ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Validation log:
- `D:\FuckAudit\Governor\Reserve Governor\poc-veto-threshold-rounding.log`

## Recommended Mitigation

Use ceil division when converting the optimistic veto threshold ratio into token units, matching both the README and the contract’s own `proposalThreshold()` rounding convention.

For example:

```solidity
uint256 vetoThresholdTok = (_vetoThreshold * pastSupply + (1e18 - 1)) / 1e18;
vetoThresholdTok = Math.max(vetoThresholdTok, 1);
```

This preserves the intended minimum-one-vote behavior while preventing fractional thresholds from resolving one vote too low.

