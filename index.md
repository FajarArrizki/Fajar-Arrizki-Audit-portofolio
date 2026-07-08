# Audit Findings Index

---

> Index of all findings from the Bitgo and Reserve Governor security audits. Organized by severity and clustered by bug type.

---

## Medium

Medium severity findings — vulnerabilities with conditional impact, requiring specific preconditions or resulting in indirect losses.

| # | Protocol | Folder | File | Cluster Bug |
|---|----------|--------|------|-------------|
| 1 | Reserve Governor | `rg-m-01` | [rg-m-01.md](Reserve%20Governor/Medium/rg-m-01/rg-m-01.md) | Governance / Threshold |
| 2 | Reserve Governor | `rg-m-02` | [rg-m-02.md](Reserve%20Governor/Medium/rg-m-02/rg-m-02.md) | Governance / Threshold |
| 3 | Reserve Governor | `rg-m-03` | [rg-m-03.md](Reserve%20Governor/Medium/rg-m-03/rg-m-03.md) | Accounting / Reward Griefing |
| 4 | Reserve Governor | `rg-m-04` | [rg-m-04.md](Reserve%20Governor/Medium/rg-m-04/rg-m-04.md) | Governance / Quorum Mismatch |

> **Medium** — 4 findings spanning: (rg-m-01) passive supply making optimistic veto threshold unreachable; (rg-m-02) passive supply raising standard proposal threshold above entire electorate; (rg-m-03) reward token unregistration + permissionless poke burning reward emission windows; (rg-m-04) undelegated deposits inflating quorum without adding votes.

---

## Low

Low severity findings — informational issues, code quality, dead code, or edge cases with minimal to no direct financial impact.

| # | Protocol | Folder | File | Cluster Bug |
|---|----------|--------|------|-------------|
| 1 | Reserve Governor | `rg-l-01` | [rg-l-01.md](Reserve%20Governor/Low/rg-l-01/rg-l-01.md) | Governance / Logic Error |
| 2 | Reserve Governor | `rg-l-02` | [rg-l-02.md](Reserve%20Governor/Low/rg-l-02/rg-l-02.md) | Accounting / Stale Pricing |
| 3 | Reserve Governor | `rg-l-03` | [rg-l-03.md](Reserve%20Governor/Low/rg-l-03/rg-l-03.md) | Math / Precision |
| 4 | Reserve Governor | `rg-l-04` | [rg-l-04.md](Reserve%20Governor/Low/rg-l-04/rg-l-04.md) | Access Control / Stale Authority |
| 5 | Reserve Governor | `rg-l-05` | [rg-l-05.md](Reserve%20Governor/Low/rg-l-05/rg-l-05.md) | ERC4626 / Interface Compliance |
| 6 | Reserve Governor | `rg-l-06` | [rg-l-06.md](Reserve%20Governor/Low/rg-l-06/rg-l-06.md) | ETH Accounting / Timelock |
| 7 | Reserve Governor | `rg-l-07` | [rg-l-07.md](Reserve%20Governor/Low/rg-l-07/rg-l-07.md) | Access Control / Insufficient Validation |
| 8 | Reserve Governor | `rg-l-08` | [rg-l-08.md](Reserve%20Governor/Low/rg-l-08/rg-l-08.md) | Accounting / Rounding |
| 9 | Reserve Governor | `rg-l-09` | [rg-l-09.md](Reserve%20Governor/Low/rg-l-09/rg-l-09.md) | State / Gas Bloat |
| 10 | Reserve Governor | `rg-l-10` | [rg-l-10.md](Reserve%20Governor/Low/rg-l-10/rg-l-10.md) | Timestamp Dependency |
| 11 | Bitgo | `b-l-01` | [b-l-01.md](Bitgo/Low/b-l-01/b-l-01.md) | Implementation Capture / Access Control |
| 12 | Bitgo | `b-l-02` | [b-l-02.md](Bitgo/Low/b-l-02/b-l-02.md) | Implementation Capture / Access Control |
| 13 | Bitgo | `b-l-03` | [b-l-03.md](Bitgo/Low/b-l-03/b-l-03.md) | Implementation Capture / Access Control |
| 14 | Bitgo | `b-l-04` | [b-l-04.md](Bitgo/Low/b-l-04/b-l-04.md) | Access Control / Approval Exploit |

> **Low** — 14 findings spanning: (rg-l-01) original optimistic proposer canceling auto-created confirmation proposal; (rg-l-02) delayed reward accounting letting new depositors capture pending rewards; (rg-l-03) veto threshold rounding down instead of up; (rg-l-04) revoked proposer still canceling succeeded proposal; (rg-l-05) maxWithdraw/maxRedeem ignoring delayed unstaking; (rg-l-06) overpaying execute trapping surplus ETH in timelock; (rg-l-07) selector-only allowlisting not constraining call arguments; (rg-l-08) reward dust becoming permanently unclaimable; (rg-l-09) unbounded proposal descriptions duplicating across confirmation transition; (rg-l-10) veto window collapsing after timestamp discontinuity; (b-l-01) WalletSimple implementation capture via public init; (b-l-02) Forwarder implementation capture via public init; (b-l-03) ForwarderV4 implementation capture granting parent and feeAddress roles; (b-l-04) ForwarderV4 ERC721 approval drain scaling linearly with collector count.

---

**Total:** 0 High | 4 Medium | 14 Low = **18 findings**
