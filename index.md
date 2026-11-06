# Audit Findings Index

---

> Index of all findings from the Reserve Governor security audit. Organized by severity and clustered by bug type.

---

## Medium

Medium severity findings — vulnerabilities with conditional impact, requiring specific preconditions or resulting in indirect losses.

| ID | Protocol | Finding | Cluster Bug |
|----|----------|---------|-------------|
| rg-m-01 | Reserve Governor | [Passive Supply Can Make the Optimistic Veto Threshold Unreachable for the Entire Delegated Electorate](Reserve%20Governor/Medium/passive-supply-can-make-the-optimistic-veto-threshold-unreachable-for-the-entire-delegated-electorate.md) | Governance / Threshold |
| rg-m-02 | Reserve Governor | [Passive Supply Can Raise the Standard Proposal Threshold Above the Entire Delegated Electorate](Reserve%20Governor/Medium/passive-supply-can-raise-the-standard-proposal-threshold-above-the-entire-delegated-electorate.md) | Governance / Threshold |
| rg-m-03 | Reserve Governor | [Unregistering a Reward Token Plus Permissionless Poke Can Burn Pending Reward Emission Windows](Reserve%20Governor/Medium/unregistering-a-reward-token-plus-permissionless-poke-can-burn-pending-reward-emission-windows.md) | Accounting / Reward Griefing |
| rg-m-04 | Reserve Governor | [Undelegated Deposits Inflate Quorum Without Adding Votes](Reserve%20Governor/Medium/undelegated-deposits-inflate-quorum-without-adding-votes.md) | Governance / Quorum Mismatch |

> **Medium** — 4 findings spanning: passive supply making the optimistic veto threshold unreachable; passive supply raising the standard proposal threshold above the entire electorate; reward token unregistration plus permissionless poke burning reward emission windows; undelegated deposits inflating quorum without adding votes.

---

## Low

Low severity findings — informational issues, code quality, dead code, or edge cases with minimal to no direct financial impact.

| ID | Protocol | Finding | Cluster Bug |
|----|----------|---------|-------------|
| rg-l-01 | Reserve Governor | [The Original Optimistic Proposer Can Cancel the Auto-Created Pending Confirmation Proposal](Reserve%20Governor/Low/the-original-optimistic-proposer-can-cancel-the-auto-created-pending-confirmation-proposal.md) | Governance / Logic Error |
| rg-l-02 | Reserve Governor | [Delayed Native Reward Accounting Lets New Depositors Capture Pending Rewards](Reserve%20Governor/Low/delayed-native-reward-accounting-lets-new-depositors-capture-pending-rewards.md) | Accounting / Stale Pricing |
| rg-l-03 | Reserve Governor | [Optimistic Veto Threshold Rounds Down Instead of Up](Reserve%20Governor/Low/optimistic-veto-threshold-rounds-down-instead-of-up.md) | Math / Precision |
| rg-l-04 | Reserve Governor | [A Revoked Optimistic Proposer Can Still Cancel a Succeeded Optimistic Proposal](Reserve%20Governor/Low/a-revoked-optimistic-proposer-can-still-cancel-a-succeeded-optimistic-proposal.md) | Access Control / Stale Authority |
| rg-l-05 | Reserve Governor | [maxWithdraw and maxRedeem Ignore Delayed Unstaking and Overstate Immediate Liquidity](Reserve%20Governor/Low/maxwithdraw-and-maxredeem-ignore-delayed-unstaking-and-overstate-immediate-liquidity.md) | ERC4626 / Interface Compliance |
| rg-l-06 | Reserve Governor | [Overpaying execute Traps Surplus ETH in the Timelock and Makes It Governance-Spendable](Reserve%20Governor/Low/overpaying-execute-traps-surplus-eth-in-the-timelock-and-makes-it-governance-spendable.md) | ETH Accounting / Timelock |
| rg-l-07 | Reserve Governor | [Selector-Only Allowlisting Cannot Constrain Optimistic Call Arguments](Reserve%20Governor/Low/selector-only-allowlisting-cannot-constrain-optimistic-call-arguments.md) | Access Control / Insufficient Validation |
| rg-l-08 | Reserve Governor | [Small Reward Dust Can Become Permanently Unclaimable](Reserve%20Governor/Low/small-reward-dust-can-become-permanently-unclaimable.md) | Accounting / Rounding |
| rg-l-09 | Reserve Governor | [Unbounded Optimistic Proposal Descriptions Persist and Duplicate Across Confirmation Transition](Reserve%20Governor/Low/unbounded-optimistic-proposal-descriptions-persist-and-duplicate-across-confirmation-transition.md) | State / Gas Bloat |
| rg-l-10 | Reserve Governor | [Optimistic Veto Window Can Collapse After Timestamp Discontinuity](Reserve%20Governor/Low/optimistic-veto-window-can-collapse-after-timestamp-discontinuity.md) | Timestamp Dependency |

> **Low** — 10 findings spanning: the original optimistic proposer canceling the auto-created pending confirmation proposal; delayed native reward accounting letting new depositors capture pending rewards; the optimistic veto threshold rounding down instead of up; a revoked optimistic proposer still canceling a succeeded proposal; maxWithdraw/maxRedeem ignoring delayed unstaking; overpaying execute trapping surplus ETH in the timelock; selector-only allowlisting not constraining call arguments; small reward dust becoming permanently unclaimable; unbounded proposal descriptions duplicating across the confirmation transition; the optimistic veto window collapsing after a timestamp discontinuity.

---

**Total:** 0 Critical | 0 High | 4 Medium | 10 Low = **14 findings**
