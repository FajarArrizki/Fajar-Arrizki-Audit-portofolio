# Passive Supply Can Make the Optimistic Veto Threshold Unreachable for the Entire Delegated Electorate

## Metadata

- **Number:** #1035
- **Severity:** Medium
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** Foundoneblock
- **Created at:** May 10, 2026 at 3:26 AM
- **Last updated:** June 2, 2026 at 5:52 AM

## Description

# Audited by [Fajar Arrizki]

Independent audit notes by Fajar Arrizki. Findings, proof-of-concept references, and impact assessments are documented for manual review.

_Note: Not all issues are guaranteed to be correct._

# Passive Supply Can Make the Optimistic Veto Threshold Unreachable for the Entire Delegated Electorate
---
**#M-73**
- Severity: Medium
- Validity: Tested
- Likelihood: Medium
- Impact: A large passive holder can increase the optimistic veto threshold through total supply without contributing any optimistic voting power, allowing fast-lane proposals to succeed even when the entire delegated veto electorate opposes them.

## Description

The optimistic lane computes its veto threshold from total historical supply, but actual veto power is counted from the separate optimistic delegation ledger.

That means a holder can remain economically large while contributing zero castable optimistic veto votes, because optimistic voting power is only created when the holder actively delegates optimistic votes.

So the system can reach a state where the full delegated veto electorate votes `Against`, but still cannot defeat the proposal because passive supply has raised the denominator too far.

### The optimistic veto threshold is derived from total supply while veto votes come from the optimistic delegation ledger

```solidity
uint256 pastSupply = token().getPastTotalSupply(snapshot);

if (pastSupply == 0) {
    return ProposalState.Canceled;
}

uint256 vetoThresholdTok = (_vetoThreshold * pastSupply) / 1e18;
vetoThresholdTok = Math.max(vetoThresholdTok, 1);

(uint256 againstVotes,,) = proposalVotes(proposalId);

if (againstVotes >= vetoThresholdTok) {
    return ProposalState.Defeated;
}
```

Source:
- `contracts/governance/ReserveOptimisticGovernor.sol:249-263`

GitHub permalink:
- `https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L249-L263`

## Root Cause

The root cause is that the fast-lane veto threshold is keyed to total supply, while veto authority is keyed to delegated optimistic votes.

```text
root cause:
- optimistic veto threshold uses total token supply at the proposal snapshot
- optimistic voting power uses a separate optimistic delegation checkpoint system
- passive holders increase the denominator even when they do not contribute any veto power
- delegated veto voters can therefore become structurally unable to defeat a proposal
```

## Impact

This issue can cause:

- fast-lane proposals to succeed even when the full delegated veto electorate opposes them,
- passive capital to harden optimistic execution without taking any active governance responsibility,
- and governance liveness to shift toward optimistic proposers whenever delegation participation lags behind economic ownership.

This is a governance-control issue with real pass/fail consequences, so Medium severity is appropriate.

## Likelihood

Likelihood is **medium**.

The path does not require privileged roles or custom token behavior. It only requires a sufficiently large passive stake to remain undelegated on the optimistic ledger while a smaller active electorate attempts to veto.

## Severity Rationale

### Why Impact is Medium

The impact is **medium** because the defect can directly change whether an optimistic proposal is vetoable, which is one of the core safety boundaries of the fast lane.

### Why it is not High

It is **not high** because the issue does not directly grant new voting power, seize funds, or bypass all governance controls. It distorts veto reachability, but still requires an existing proposal and an unfavorable stake/delegation distribution.

## Proof of Concept

### What the test proves

The standalone PoC demonstrates:

1. a large passive holder can own most of the supply without any optimistic delegation,
2. a smaller set of active veto voters can hold all delegated optimistic votes,
3. the optimistic veto threshold still scales with the passive holder’s supply,
4. the full delegated veto electorate can vote `Against`,
5. yet the proposal still succeeds because the computed threshold remains above all available delegated veto votes.

## Proof of Concept

### PocNewBase.t.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { Test } from "forge-std/Test.sol";

import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import { IERC5805 } from "@openzeppelin/contracts/interfaces/IERC5805.sol";
import { Checkpoints } from "@openzeppelin/contracts/utils/structs/Checkpoints.sol";
import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import { IReserveOptimisticGovernorDeployer } from "@interfaces/IDeployer.sol";
import { IOptimisticSelectorRegistry } from "@interfaces/IOptimisticSelectorRegistry.sol";
import { IOptimisticVotes } from "@interfaces/IOptimisticVotes.sol";
import { IReserveOptimisticGovernor } from "@interfaces/IReserveOptimisticGovernor.sol";
import { IRoleRegistry } from "@interfaces/IRoleRegistry.sol";

import { ReserveOptimisticGovernorDeployer } from "@src/Deployer.sol";
import { Guardian } from "@src/Guardian.sol";
import { ReserveOptimisticGovernanceVersionRegistry } from "@src/VersionRegistry.sol";
import { OptimisticSelectorRegistry } from "@governance/OptimisticSelectorRegistry.sol";
import { ReserveOptimisticGovernor } from "@governance/ReserveOptimisticGovernor.sol";
import { TimelockControllerOptimistic } from "@governance/TimelockControllerOptimistic.sol";
import { RewardTokenRegistry } from "@staking/RewardTokenRegistry.sol";
import { StakingVault } from "@staking/StakingVault.sol";
import { UnstakingManager } from "@staking/UnstakingManager.sol";
import { CANCELLER_ROLE, EXECUTOR_ROLE, OPTIMISTIC_GUARDIAN_ROLE, PROPOSER_ROLE } from "@utils/Constants.sol";
import { Versioned } from "@utils/Versioned.sol";

import { MockERC20 } from "@mocks/MockERC20.sol";
import { MockRoleRegistry } from "@mocks/MockRoleRegistry.sol";

contract NoopTarget {
    uint256 public calls;

    function touch() external {
        calls++;
    }
}

contract MutableERC20 is ERC20 {
    bool public paused;
    bool public revertBalance;
    mapping(address => bool) public blacklisted;

    constructor(string memory name_, string memory symbol_) ERC20(name_, symbol_) { }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function setPaused(bool paused_) external {
        paused = paused_;
    }

    function setBlacklisted(address account, bool blacklisted_) external {
        blacklisted[account] = blacklisted_;
    }

    function setRevertBalance(bool revert_) external {
        revertBalance = revert_;
    }

    function balanceOf(address account) public view override returns (uint256) {
        require(!revertBalance, "balanceOf reverted");
        return super.balanceOf(account);
    }

    function _update(address from, address to, uint256 value) internal override {
        require(!paused, "paused");
        require(!blacklisted[from] && !blacklisted[to], "blacklisted");
        super._update(from, to, value);
    }
}

contract PartialSupplyOptimisticToken is ERC20, IERC5805, IOptimisticVotes {
    mapping(address => uint256) public optimisticVotes;
    bool public revertPastSupply;

    constructor() ERC20("Partial Optimistic", "PART") { }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
        optimisticVotes[to] += amount;
    }

    function setRevertPastSupply(bool revert_) external {
        revertPastSupply = revert_;
    }

    function clock() external view returns (uint48) {
        return uint48(block.timestamp);
    }

    function CLOCK_MODE() external pure returns (string memory) {
        return "mode=timestamp";
    }

    function getVotes(address account) external view returns (uint256) {
        return balanceOf(account);
    }

    function getPastVotes(address account, uint256) external view returns (uint256) {
        return balanceOf(account);
    }

    function getPastTotalSupply(uint256) external view returns (uint256) {
        require(!revertPastSupply, "past supply unavailable");
        return totalSupply();
    }

    function delegates(address account) external view returns (address) {
        return account;
    }

    function delegate(address) external { }

    function delegateBySig(address, uint256, uint256, uint8, bytes32, bytes32) external { }

    function getPastOptimisticVotes(address account, uint256) external view returns (uint256) {
        return optimisticVotes[account];
    }

    function numOptimisticCheckpoints(address) external pure returns (uint32) {
        return 0;
    }

    function optimisticCheckpoints(address, uint32) external pure returns (Checkpoints.Checkpoint208 memory) {
        return Checkpoints.Checkpoint208({ _key: 0, _value: 0 });
    }
}

contract MutableDeployer is Versioned, IReserveOptimisticGovernorDeployer {
    address public override versionRegistry;
    address public override rewardTokenRegistry;
    address public override guardian;
    address public override stakingVaultImpl;
    address public override governorImpl;
    address public override timelockImpl;

    constructor(address stakingVaultImpl_, address governorImpl_, address timelockImpl_) {
        stakingVaultImpl = stakingVaultImpl_;
        governorImpl = governorImpl_;
        timelockImpl = timelockImpl_;
    }

    function setStakingVaultImpl(address stakingVaultImpl_) external {
        stakingVaultImpl = stakingVaultImpl_;
    }

    function deployWithNewStakingVault(BaseDeploymentParams calldata, NewStakingVaultParams calldata, bytes32)
        external
        pure
        returns (address, address, address, address)
    {
        revert("not used");
    }

    function deployWithExistingStakingVault(BaseDeploymentParams calldata, address, bytes32)
        external
        pure
        returns (address, address, address)
    {
        revert("not used");
    }
}

contract MutableVersionedDeployer is MutableDeployer {
    string private constant VERSION_ONE_ONE = "1.0.1";

    constructor(address stakingVaultImpl_, address governorImpl_, address timelockImpl_)
        MutableDeployer(stakingVaultImpl_, governorImpl_, timelockImpl_)
    { }

    function version() public pure override returns (string memory) {
        return VERSION_ONE_ONE;
    }
}

contract AlternateVersionDeployer is ReserveOptimisticGovernorDeployer {
    string private constant VERSION_TWO = "2.0.0";

    constructor(
        address versionRegistry_,
        address rewardTokenRegistry_,
        address guardian_,
        address stakingVaultImpl_,
        address governorImpl_,
        address timelockImpl_,
        address selectorRegistryImpl_
    )
        ReserveOptimisticGovernorDeployer(
            versionRegistry_, rewardTokenRegistry_, guardian_, stakingVaultImpl_, governorImpl_, timelockImpl_, selectorRegistryImpl_
        )
    {
    }

    function version() public pure override returns (string memory) {
        return VERSION_TWO;
    }
}

abstract contract PocTestNewBase is Test {
    MockERC20 internal underlying;
    StakingVault internal stakingVault;
    OptimisticSelectorRegistry internal registry;
    Guardian internal guardianContract;
    ReserveOptimisticGovernorDeployer internal deployer;
    ReserveOptimisticGovernor internal governor;
    TimelockControllerOptimistic internal timelock;
    ReserveOptimisticGovernanceVersionRegistry internal versionRegistry;
    RewardTokenRegistry internal rewardTokenRegistry;
    MockRoleRegistry internal roleRegistry;

    address internal alice = makeAddr("alice");
    address internal bob = makeAddr("bob");
    address internal carol = makeAddr("carol");
    address internal guardian = makeAddr("guardian");
    address internal additionalGuardian = makeAddr("additionalGuardian");
    address internal optimisticGuardianManager = makeAddr("optimisticGuardianManager");
    address internal optimisticGuardian = makeAddr("optimisticGuardian");
    address internal badGuardian = makeAddr("badGuardian");
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
    uint256 internal constant ALICE_STAKE = 400_000e18;
    uint256 internal constant BOB_STAKE = 400_000e18;
    uint256 internal constant CAROL_STAKE = 200_000e18;

    function _deploy(uint256 unstakingDelay) internal {
        underlying = new MockERC20("Underlying Token", "UNDL");
        roleRegistry = new MockRoleRegistry(address(this));
        versionRegistry = new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

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
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData = new IOptimisticSelectorRegistry.SelectorData[](1);
        selectorData[0] = IOptimisticSelectorRegistry.SelectorData(address(underlying), transferSelectors);

        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams = _baseParams(selectorData, optimisticProposers, TIMELOCK_DELAY);

        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory vaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: new address[](0),
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: unstakingDelay
            });

        (address vaultAddr, address governorAddr, address timelockAddr, address selectorAddr) =
            deployer.deployWithNewStakingVault(baseParams, vaultParams, bytes32(0));

        stakingVault = StakingVault(vaultAddr);
        governor = ReserveOptimisticGovernor(payable(governorAddr));
        timelock = TimelockControllerOptimistic(payable(timelockAddr));
        registry = OptimisticSelectorRegistry(selectorAddr);

        _setupVoter(alice, ALICE_STAKE);
        _setupVoter(bob, BOB_STAKE);
        _setupVoter(carol, CAROL_STAKE);
        vm.warp(block.timestamp + 12 hours);
    }

    function _baseParams(
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData,
        address[] memory optimisticProposers,
        uint256 timelockDelay
    ) internal view returns (IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory) {
        address[] memory additionalGuardians = new address[](1);
        additionalGuardians[0] = additionalGuardian;

        return IReserveOptimisticGovernorDeployer.BaseDeploymentParams({
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
            additionalGuardians: additionalGuardians,
            timelockDelay: timelockDelay,
            proposalThrottleCapacity: PROPOSAL_THROTTLE_CAPACITY
        });
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

    function _passAndQueue(address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
        internal
        returns (bytes32 descriptionHash)
    {
        vm.prank(alice);
        uint256 proposalId = governor.propose(targets, values, calldatas, description);
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

    function _registerOptimisticSelector(address target, bytes4 selector) internal {
        bytes4[] memory selectors = new bytes4[](1);
        selectors[0] = selector;
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData = new IOptimisticSelectorRegistry.SelectorData[](1);
        selectorData[0] = IOptimisticSelectorRegistry.SelectorData(target, selectors);
        vm.prank(address(timelock));
        registry.registerSelectors(selectorData);
    }

    function _simpleBaseParams()
        internal
        view
        returns (IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams)
    {
        address[] memory optimisticProposers = new address[](2);
        optimisticProposers[0] = optimisticProposer;
        optimisticProposers[1] = optimisticProposer2;

        IOptimisticSelectorRegistry.SelectorData[] memory selectorData = new IOptimisticSelectorRegistry.SelectorData[](0);
        baseParams = _baseParams(selectorData, optimisticProposers, TIMELOCK_DELAY);
    }

    function _deployExistingTokenSystem(address token) internal {
        underlying = new MockERC20("Dummy Underlying", "DUM");
        roleRegistry = new MockRoleRegistry(address(this));
        versionRegistry = new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

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

        (address governorAddr, address timelockAddr, address selectorAddr) =
            deployer.deployWithExistingStakingVault(_simpleBaseParams(), token, bytes32(uint256(12345)));

        governor = ReserveOptimisticGovernor(payable(governorAddr));
        timelock = TimelockControllerOptimistic(payable(timelockAddr));
        registry = OptimisticSelectorRegistry(selectorAddr);
        vm.warp(block.timestamp + 12 hours);
    }

    function _deployWithTimelockDelay(uint256 timelockDelay) internal {
        underlying = new MockERC20("Underlying Token", "UNDL");
        roleRegistry = new MockRoleRegistry(address(this));
        versionRegistry = new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

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

        address[] memory optimisticProposers = new address[](1);
        optimisticProposers[0] = optimisticProposer;
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData = new IOptimisticSelectorRegistry.SelectorData[](0);
        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams =
            _baseParams(selectorData, optimisticProposers, timelockDelay);
        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory vaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: IERC20Metadata(address(underlying)),
                rewardTokens: new address[](0),
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: 0
            });

        (address vaultAddr, address governorAddr, address timelockAddr, address selectorAddr) =
            deployer.deployWithNewStakingVault(baseParams, vaultParams, bytes32(uint256(222)));

        stakingVault = StakingVault(vaultAddr);
        governor = ReserveOptimisticGovernor(payable(governorAddr));
        timelock = TimelockControllerOptimistic(payable(timelockAddr));
        registry = OptimisticSelectorRegistry(selectorAddr);
        _setupVoter(alice, ALICE_STAKE);
        _setupVoter(bob, BOB_STAKE);
        vm.warp(block.timestamp + 12 hours);
    }

    function _deployWithUnderlying(IERC20Metadata token, uint256 unstakingDelay) internal {
        roleRegistry = new MockRoleRegistry(address(this));
        versionRegistry = new ReserveOptimisticGovernanceVersionRegistry(roleRegistry);
        rewardTokenRegistry = new RewardTokenRegistry(roleRegistry);

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

        address[] memory optimisticProposers = new address[](1);
        optimisticProposers[0] = optimisticProposer;
        IOptimisticSelectorRegistry.SelectorData[] memory selectorData = new IOptimisticSelectorRegistry.SelectorData[](0);
        IReserveOptimisticGovernorDeployer.BaseDeploymentParams memory baseParams =
            _baseParams(selectorData, optimisticProposers, TIMELOCK_DELAY);
        IReserveOptimisticGovernorDeployer.NewStakingVaultParams memory vaultParams =
            IReserveOptimisticGovernorDeployer.NewStakingVaultParams({
                underlying: token,
                rewardTokens: new address[](0),
                rewardHalfLife: REWARD_HALF_LIFE,
                unstakingDelay: unstakingDelay
            });

        (address vaultAddr, address governorAddr, address timelockAddr, address selectorAddr) =
            deployer.deployWithNewStakingVault(baseParams, vaultParams, bytes32(uint256(333)));

        stakingVault = StakingVault(vaultAddr);
        governor = ReserveOptimisticGovernor(payable(governorAddr));
        timelock = TimelockControllerOptimistic(payable(timelockAddr));
        registry = OptimisticSelectorRegistry(selectorAddr);
        vm.warp(block.timestamp + 12 hours);
    }
}
```

### PocF01_OptimisticVetoPassiveSupply.t.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IERC20Metadata } from "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";

import { MockERC20 } from "@mocks/MockERC20.sol";
import { NoopTarget, PocTestNewBase } from "./PocTestNewBase.t.sol";

contract PocF01_OptimisticVetoPassiveSupply_Test is PocTestNewBase {
    uint256 internal constant PASSIVE_ALICE = 900e18;
    uint256 internal constant DELEGATED_BOB = 50e18;
    uint256 internal constant DELEGATED_CAROL = 50e18;

    function setUp() public {
        underlying = new MockERC20("Underlying Token", "UNDL");
        _deployWithUnderlying(IERC20Metadata(address(underlying)), 0);

        _depositPassive(alice, PASSIVE_ALICE);
        _depositAndDelegateOptimistic(bob, DELEGATED_BOB);
        _depositAndDelegateOptimistic(carol, DELEGATED_CAROL);

        vm.warp(block.timestamp + 12 hours);
    }

    function test_F01_passiveSupplyInflatesOptimisticVetoThresholdAboveEntireDelegatedElectorate() public {
        NoopTarget target = new NoopTarget();
        _registerOptimisticSelector(address(target), target.touch.selector);

        (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
            _singleCall(address(target), 0, abi.encodeCall(target.touch, ()));

        vm.prank(optimisticProposer);
        uint256 proposalId = governor.proposeOptimistic(targets, values, calldatas, "passive supply veto inflation");

        _warpToActive(proposalId);

        uint256 snapshot = governor.proposalSnapshot(proposalId);
        uint256 pastSupply = stakingVault.getPastTotalSupply(snapshot);
        uint256 bobVotes = stakingVault.getPastOptimisticVotes(bob, snapshot);
        uint256 carolVotes = stakingVault.getPastOptimisticVotes(carol, snapshot);
        uint256 totalDelegatedVetoVotes = bobVotes + carolVotes;
        uint256 vetoThresholdTok = (governor.vetoThreshold(proposalId) * pastSupply) / 1e18;
        if (vetoThresholdTok == 0) vetoThresholdTok = 1;

        assertEq(pastSupply, PASSIVE_ALICE + DELEGATED_BOB + DELEGATED_CAROL);
        assertEq(totalDelegatedVetoVotes, DELEGATED_BOB + DELEGATED_CAROL);
        assertGt(vetoThresholdTok, totalDelegatedVetoVotes, "passive supply should lift veto threshold above all delegated veto votes");

        vm.prank(bob);
        governor.castVote(proposalId, 0);
        vm.prank(carol);
        governor.castVote(proposalId, 0);

        (uint256 againstVotes,,) = governor.proposalVotes(proposalId);
        assertEq(againstVotes, totalDelegatedVetoVotes, "entire delegated electorate vetoed");
        assertLt(againstVotes, vetoThresholdTok, "veto still cannot reach inflated threshold");

        _warpPastDeadline(proposalId);
        assertEq(uint256(governor.state(proposalId)), uint256(0x4), "proposal still succeeds despite unanimous delegated veto");
    }

    function _depositPassive(address voter, uint256 amount) internal {
        underlying.mint(voter, amount);
        vm.startPrank(voter);
        underlying.approve(address(stakingVault), amount);
        stakingVault.deposit(amount, voter);
        vm.stopPrank();
    }

    function _depositAndDelegateOptimistic(address voter, uint256 amount) internal {
        underlying.mint(voter, amount);
        vm.startPrank(voter);
        underlying.approve(address(stakingVault), amount);
        stakingVault.depositAndDelegate(amount, address(0), voter);
        vm.stopPrank();
    }
}
```

### Output Test
```text
[PASS] test_F01_passiveSupplyInflatesOptimisticVetoThresholdAboveEntireDelegatedElectorate()
```

## Recommended Mitigation
- Base the optimistic veto threshold on optimistic delegated supply rather than total supply.
- Alternatively, require the optimistic lane to use a denominator that reflects only castable optimistic voting power.
- If total supply must remain the denominator, explicitly require automatic optimistic self-delegation on deposit so passive supply cannot silently dilute veto reachability.

