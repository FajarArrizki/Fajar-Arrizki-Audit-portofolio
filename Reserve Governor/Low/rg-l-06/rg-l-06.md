# Overpaying execute Traps Surplus ETH in the Timelock and Makes It Governance-Spendable

## Metadata

- **Number:** #836
- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** Foundoneblock
- **Created at:** May 9, 2026 at 6:29 AM
- **Last updated:** June 2, 2026 at 5:44 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Overpaying execute Traps Surplus ETH in the Timelock and Makes It Governance-Spendable
---
**#M-40**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: A caller can overpay `execute(...)`, trap surplus ETH inside the timelock, and later governance can redirect that stranded balance to arbitrary recipients even though the original proposal never referenced that extra value.

## Description

Optimistic and pessimistic execution both forward ETH through the timelock, but the system never reconciles `msg.value` against the sum of `values[]`.

For the optimistic path, `_executeOperations()` forwards the full `msg.value` into `executeBatchBypass(...)`.

If the batch itself uses zero ETH, the surplus is not refunded. It simply remains in the timelock balance.

That trapped ETH is not isolated.

Any later successful governance execution can spend it by proposing a nonzero `values[i]` to an arbitrary target.

So an overpaid execution silently converts accidental or malicious overpayment into future governance-controlled treasury value.

### Optimistic execution forwards the entire `msg.value`

```solidity
function _executeOperations(
    uint256 proposalId,
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory calldatas,
    bytes32 descriptionHash
) internal override(GovernorUpgradeable, GovernorTimelockControlUpgradeable) {
    if (_isOptimistic(proposalId)) {
        // optimistic case: execute immediately

        _timelock().executeBatchBypass{ value: msg.value }(
            targets, values, calldatas, 0, bytes20(address(this)) ^ descriptionHash
        );
    } else {
        // pessimistic case: execute through timelock

        super._executeOperations(proposalId, targets, values, calldatas, descriptionHash);
    }
}
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:345-362`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L345-L362`

No equality check is enforced between forwarded ETH and the economic value encoded in `values[]`.

### The timelock bypass path marks ready and executes, but never refunds surplus ETH

```solidity
function executeBatchBypass(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata payloads,
    bytes32 predecessor,
    bytes32 salt
) public payable onlyRole(PROPOSER_ROLE) {
    bytes32 id = hashOperationBatch(targets, values, payloads, predecessor, salt);

    TimelockControllerStorage storage $ = _getTimelockControllerStorage();

    // mark Ready
    require($._timestamps[id] == 0, TimelockControllerOptimistic__OperationConflict());
    $._timestamps[id] = block.timestamp;

    // check caller has EXECUTOR_ROLE and execute
    executeBatch(targets, values, payloads, predecessor, salt);
}
```

Source:
- `contracts/governance/TimelockControllerOptimistic.sol:76-92`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/TimelockControllerOptimistic.sol#L76-L92`

There is no refund path here and no explicit sum check for batch value consumption.

## Root Cause

The root cause is that governance execution treats incoming ETH as ambient timelock funding instead of exact proposal-scoped payment.

```text
root cause:
- governor forwards full msg.value into timelock execution
- timelock does not reconcile msg.value against values[]
- unused ETH remains on the timelock balance
- later governance executions can spend that trapped ETH with unrelated proposals
```

## Impact

This issue can cause:

- accidental ETH overpayment during execution to become trapped,
- trapped ETH to become silently governance-spendable by unrelated later proposals,
- executors and operators to misattribute proposal value consumption,
- and governance to gain an implicit ETH treasury that was never part of the original proposal payload.

This is a treasury/accounting integrity issue rather than an unprivileged direct theft path, so Medium severity is appropriate.

## Likelihood

Likelihood is **medium**.

The path requires either:

- an overpaying executor,
- or intentional overpayment by a caller,
- and later governance control to redirect the trapped ETH.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because surplus ETH can be trapped and then later redirected by governance to arbitrary recipients. It is real value flow, not just a cosmetic accounting mismatch.

### Why it is not High

It is **not high** because the exploit still depends on governance execution authority; an arbitrary external attacker cannot directly drain the timelock without a successful proposal path.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. a standard proposal is queued with `values = [0]`,
2. execution is intentionally overpaid with `1 ether`,
3. the proposal executes successfully,
4. the full surplus remains trapped on the timelock balance,
5. a later proposal sends that same trapped ETH to Bob,
6. proving the surplus became unrelated governance-spendable value.

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

###  Overpaying execute Traps Surplus ETH in the Timelock and Makes It Governance-Spendable

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;
pragma abicoder v2;

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";

import { PocNewBaseNewVault } from "./PocNewBase.t.sol";

contract PocTimelockExcessEthTrapTest is PocNewBaseNewVault {
    function test_poc_excessMsgValueGetsTrappedAndLaterSpentFromTimelock() public {
        uint256 trappedEth = 1 ether;

        (address[] memory trapTargets, uint256[] memory trapValues, bytes[] memory trapCalldatas) =
            _singleCall(bob, 0, bytes(""));

        (uint256 trapProposalId, bytes32 trapDescriptionHash) = _proposePassAndQueueStandard(
            trapTargets, trapValues, trapCalldatas, "queue zero-value call then overpay execute"
        );

        vm.warp(block.timestamp + TIMELOCK_DELAY + 1);
        governor.execute{ value: trappedEth }(trapTargets, trapValues, trapCalldatas, trapDescriptionHash);

        assertEq(uint256(governor.state(trapProposalId)), uint256(IGovernor.ProposalState.Executed));
        assertEq(address(timelock).balance, trappedEth, "surplus ETH should remain trapped in timelock");

        uint256 bobEthBefore = bob.balance;
        (address[] memory drainTargets, uint256[] memory drainValues, bytes[] memory drainCalldatas) =
            _singleCall(bob, trappedEth, bytes(""));

        (, bytes32 drainDescriptionHash) =
            _proposePassAndQueueStandard(drainTargets, drainValues, drainCalldatas, "spend previously trapped ETH");

        vm.warp(block.timestamp + TIMELOCK_DELAY + 1);
        governor.execute(drainTargets, drainValues, drainCalldatas, drainDescriptionHash);

        assertEq(bob.balance, bobEthBefore + trappedEth, "later governance execution can spend trapped ETH");
        assertEq(address(timelock).balance, 0, "timelock surplus should be fully drainable by later proposal");
    }
}
```

### Test command

```powershell
docker run --rm --entrypoint sh -v "D:\FuckAudit\Governor\Reserve Governor:/work" -w /work ghcr.io/foundry-rs/foundry:latest -lc "forge test --match-path test/poc_new/PocTimelockExcessEthTrap.t.sol -vvv"
```

### Test output

```text
[PASS] test_poc_excessMsgValueGetsTrappedAndLaterSpentFromTimelock()
```

