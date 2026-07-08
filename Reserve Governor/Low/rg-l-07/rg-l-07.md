# Selector-Only Allowlisting Cannot Constrain Optimistic Call Arguments

## Metadata

- **Number:** #840
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 6:34 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Selector-Only Allowlisting Cannot Constrain Optimistic Call Arguments
---
**#M-45**
- Severity: Medium
- Validity: Tested
- Likelihood: High
- Impact: Once a target selector is allowlisted, optimistic proposals can supply arbitrary recipient and amount parameters to that function, so the registry cannot constrain the economic meaning of approved optimistic calls.

## Description

The optimistic selector registry is parameter-blind.

It checks only:

- the target address,
- and the first 4 bytes of calldata.

So allowlisting a selector such as `disburse(address,uint256)` does **not** constrain:

- who receives funds,
- how much is transferred,
- or any other economically critical arguments.

That means “safe selector” approval is much weaker than it appears.

The registry is not approving a policy like “small treasury disbursement to a known recipient.”

It is approving “any call whose first 4 bytes equal this function selector.”

### The registry stores selectors only, not argument policies

```solidity
function isAllowed(address target, bytes4 selector) external view returns (bool) {
    return _allowedSelectors[target].contains(bytes32(selector));
}
```

Source:
- `contracts/governance/OptimisticSelectorRegistry.sol:68-70`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/OptimisticSelectorRegistry.sol#L68-L70`

### Proposal admission checks only the 4-byte prefix

```solidity
require(
    target.code.length != 0 && proposal.calldatas[i].length >= 4
        && selectorRegistry.isAllowed(target, bytes4(proposal.calldatas[i])),
    IReserveOptimisticGovernor.OptimisticGovernor__InvalidCall(target, proposal.calldatas[i])
);
```

Source:
- `contracts/governance/lib/ProposalLib.sol:54-61`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L54-L61`

No argument whitelisting, recipient guard, amount cap, or calldata-shape policy exists here.

## Root Cause

The root cause is that the optimistic-call policy is defined at selector granularity only, even though many selectors expose economically unconstrained parameters.

```text
root cause:
- selector registry approves target + 4-byte selector only
- optimistic proposal admission validates only that coarse tuple
- full calldata remains attacker-controlled beyond the first 4 bytes
- any economically sensitive parameters inside the allowed function stay unconstrained
```

## Impact

This issue can cause:

- allowlisted treasury or asset-moving functions to be called with arbitrary recipients and amounts,
- operator assumptions about “safe approved functions” to fail,
- and optimistic execution to admit economically dangerous parameterizations of superficially benign selectors.

This is a policy-integrity issue rather than a direct bypass of the current access checks, so Medium severity is appropriate.

## Likelihood

Likelihood is **high**.

This is inherent to the design whenever a target selector with meaningful parameters is allowlisted.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the bug widens every allowlisted selector into an argument-unbounded call surface. The registry cannot express the intended economic constraints of the operation.

### Why it is not High

It is **not high** because the issue still depends on governance choosing to allowlist a target selector and an optimistic proposer being able to use it; it is not an arbitrary outsider bypass of all policy.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. a treasury-like target holds `25_000e18` underlying,
2. governance allowlists only the `disburse(address,uint256)` selector,
3. no recipient or amount policy is encoded in the registry,
4. an optimistic proposer submits the same allowlisted selector with Alice as recipient and the full balance as amount,
5. the proposal is accepted and executed successfully,
6. proving selector-only approval cannot constrain the call’s economic parameters.

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

### Selector-Only Allowlisting Cannot Constrain Optimistic Call Arguments

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IOptimisticSelectorRegistry } from "@interfaces/IOptimisticSelectorRegistry.sol";

import { ParameterizedDisburserMock } from "@mocks/ParameterizedDisburserMock.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocSelectorOnlyAllowlistPermitsArbitraryArgumentsTest is PocNewBaseNewVault {
    function test_poc_selectorAllowlistCannotConstrainRecipientOrAmount() public {
        ParameterizedDisburserMock treasury = new ParameterizedDisburserMock(underlying);
        underlying.mint(address(treasury), 25_000e18);

        bytes4[] memory selectors = new bytes4[](1);
        selectors[0] = treasury.disburse.selector;
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData =
            new IOptimisticSelectorRegistry.SelectorData[](1);
        selectorData[0] = IOptimisticSelectorRegistry.SelectorData(address(treasury), selectors);

        (address[] memory allowTargets, uint256[] memory allowValues, bytes[] memory allowCalldatas) = _singleCall(
            address(registry), 0, abi.encodeCall(registry.registerSelectors, (selectorData))
        );
        (, bytes32 allowDescriptionHash) =
            _proposePassAndQueueStandard(allowTargets, allowValues, allowCalldatas, "whitelist disburse selector only");
        vm.warp(block.timestamp + TIMELOCK_DELAY + 1);
        governor.execute(allowTargets, allowValues, allowCalldatas, allowDescriptionHash);

        uint256 aliceUnderlyingBefore = underlying.balanceOf(alice);
        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) = _singleCall(
            address(treasury), 0, abi.encodeCall(treasury.disburse, (alice, 25_000e18))
        );

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(
            targets,
            values,
            calldatas,
            "selector allowlist accepts arbitrary recipient and amount parameters"
        );

        _warpToActive(proposalId);
        _warpPastDeadline(proposalId);
        governor.execute(
            targets,
            values,
            calldatas,
            keccak256(bytes("selector allowlist accepts arbitrary recipient and amount parameters"))
        );

        assertEq(
            underlying.balanceOf(alice),
            aliceUnderlyingBefore + 25_000e18,
            "selector-only allowlist cannot constrain the economic parameters of the call"
        );
    }
}
```

### Test command

```powershell
docker run --rm --entrypoint sh -v "D:\FuckAudit\Governor\Reserve Governor:/work" -w /work ghcr.io/foundry-rs/foundry:latest -lc "forge test --match-path test/poc_new/PocSelectorOnlyAllowlistPermitsArbitraryArguments.t.sol -vvv"
```

### Test output

```text
[PASS] test_poc_selectorAllowlistCannotConstrainRecipientOrAmount()
```

