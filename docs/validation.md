# Sanitized validation summary

This file records what has been proven live and offline, by version. Identifiers are omitted.
Dated sanitized records: [2026-09-06 standalone walkthrough](LIVE-TEST-2026-09-06.md) and
[2026-09-27 new-tenant run with the first protected-policy write](LIVE-TEST-2026-09-27.md).

## 1.0 — live validation (2026-08-24)

Validated against isolated live Azure resources on 2026-08-24. Environment identifiers and job IDs are intentionally omitted.

## Control-plane result

| Scope | Phase | Observed result |
|---|---|---|
| Resource group | Audit | One exact-name policy reported `WouldEnableTierRecommended`; writes `0`; errors `0` |
| Resource group | Apply | `DoNotTier` changed to `TierRecommended`; writes `1`; errors `0` |
| Resource group | Idempotence | `AlreadyCompliant`; writes `0`; errors `0` |
| Subscription | Audit | Two exact-name policies found: one compliant and one would enable; writes `0`; errors `0` |
| Subscription | Apply | One remained compliant and one changed to `TierRecommended`; writes `1`; errors `0` |
| Subscription | Idempotence | Both `AlreadyCompliant`; writes `0`; errors `0` |

Azure Activity Log attributed both backup-policy writes to the Automation Account's system-assigned managed identity.

## Test fixture

- Two GeoRedundant Recovery Services vaults in one isolated resource group.
- One custom Azure VM V1/daily policy per vault.
- Both policies began at `DoNotTier` and had zero protected items.
- The managed identity had subscription discovery reads and backup-policy writer access only at the isolated resource group.
- No subscription-wide backup-policy writer assignment was used.
- No recurring Automation schedule was created.

The final policy state was `TierRecommended` in both vaults. The runbook's schedule and retention comparisons passed after each write.

## Default policies observed

Each new vault also contained service-created policies resembling:

| Policy type | Shape | Initial tiering state |
|---|---|---|
| Default Azure VM policy | V1 daily | No tiering object returned |
| Enhanced Azure VM policy | V2 hourly | No tiering object returned |
| Hourly log backup policy | Azure Workload / SQL | No tiering object returned |

The validation jobs used an exact custom policy-name filter, so none of the service-created default policies was changed. Names of default policies should not be treated as a stable API contract.

## What was not proven

- Movement of a mature recovery point into Vault Archive.
- V2/hourly policy mutation.
- A policy protecting real workloads. (A single-item V1 daily policy protecting a disposable test VM
  was first written live on 2026-09-27; production workloads remain unproven. See
  [2026-09-27](#11--new-tenant-and-first-protected-policy-write-2026-09-27).)
- SQL Server or SAP HANA `TierAfter` remediation.
- Resource Guard/MUA flows.
- Cross-tenant execution.

This evidence supports controlled canary use; it is not blanket certification for every Recovery Services backup-policy shape.


## 1.0 — live audit against the same fixture (2026-08-25)

An unfiltered resource-group audit (`Apply=false`, vault filter only) of the 1.0 runbook classified
the vault's four policies as follows:

| Policy | Shape | 1.0 result |
|---|---|---|
| Service-created default VM policy | V1, daily, 30-day retention, no monthly/yearly | `WouldEnableTierRecommended` (a write candidate) |
| Service-created enhanced VM policy | V2, hourly, no monthly/yearly | `WouldEnableTierRecommended` (a write candidate) |
| Service-created SQL log policy | AzureWorkload | `SkippedUnsupportedWorkload` |
| Canary policy | V1 daily with monthly + yearly | `AlreadyCompliant` |

The default policy was listed **before** the canary policy, so an unfiltered apply would have
attempted it first and — because 1.0 isolates errors per vault — a rejection would have skipped the
canary. This is the live counterpart of harness scenario S21 and the reason 1.1 adds
`SkippedNoArchiveEligibility` and per-policy isolation.

**Apply against the daily-only default policy (1.0, exact filter, 2026-08-25):** ARM rejected the
full-policy PUT with HTTP 400 `BMSUserErrorInvalidPolicyInput` ("Input for create or update policy
is not in proper format. Please check format of parameters like schedule time, schedule days,
retention time and retention days"). The 1.0 runbook recorded an `Error` row with `policy: null`
and the job failed; the policy was unchanged. Unfiltered, that failure would have abandoned the
eligible canary policy listed after it in the same vault. 1.1 never submits that PUT.

## 1.1 — offline verification (2026-08-25)

`tests/BehaviorHarness.ps1` (45 scenarios) executes the unmodified runbook under PowerShell 7.4 with a mocked
ARM transport and asserts per-policy actions, write counters, PUT bodies, abort reasons and the
absence of ARM calls for parameter-validation failures. Scenarios cover: classification of every
mode, ZRS and workload skips, pagination, `429` and `Retry-After`, `401` token refresh, `202` with
`Azure-AsyncOperation` followed to `Succeeded`/`Failed`, verification that never converges
(`WriteOutcomeUnknown`), a lost PUT response reconciled by re-read, tag preservation, member-order
changes, daily-only and 8-vs-9-month eligibility, protected-item limits, `MaxChanges`,
`ExpectedMatches`, whitespace filters, subscription-scope resource-group rejection and per-policy
error isolation. The harness runs in CI on every push and pull request.

## 1.1 — live qualification (2026-08-25)

Run against the retained 2026-08-24 fixture as an **additional** runbook (`Enable-SmartTiering-v11`,
linked to the `PowerShell74` runtime environment; the 1.0 runbook was left untouched). The published
content hash equalled the repository file's SHA-256 (`2cef45ac…`). Eight jobs, 3–5 seconds each.

| Step | Job parameters | Result |
|---|---|---|
| Unfiltered audit | RG scope, vault filter only, `Apply=false` | `DefaultPolicy` and `EnhancedPolicy` → `SkippedNoArchiveEligibility` (horizon 0); SQL log policy → `SkippedUnsupportedWorkload`; canary → `AlreadyCompliant` (horizon 24); `errors=0` |
| Seed | `az rest` PUT of the canary policy with `DoNotTier` | mode read back as `DoNotTier` |
| Exact audit | `VaultName`+`PolicyName`, `Apply=false` | `WouldEnableTierRecommended`, `candidates=1`, writes `0` |
| Exact apply | same, `Apply=true` (defaults: `MaxChanges=1`, `MaxProtectedItemsPerPolicy=0`) | `EnabledAndVerified`, `operationStatus=HTTP 200` (synchronous), `writesSubmitted=1`, `policiesWritten=1`, `errors=0` |
| Idempotent apply | same again | `AlreadyCompliant`, `candidates=0`, `policiesWritten=0`, job Completed |
| Post-state diff | canary policy before seed vs after apply, excluding `tieringPolicy` | identical (schedule, retention, `instantRpRetentionRangeInDays`, `timeZone`, `location`) |
| Guard: whitespace `VaultName` | `VaultName="   "`, `Apply=true` | job **Failed** at parameter validation before any ARM call |
| Guard: unfiltered apply | no filters, `Apply=true`, no opt-in | job **Failed** at parameter validation before any ARM call |
| Guard: misspelled `PolicyName` | exact vault, wrong policy, `Apply=true` | job **Failed** with `abortReason=NoPolicyMatched`, `writesSubmitted=0` |
| Opt-in unfiltered apply | `AllowUnfilteredApply=true`, `Apply=true`, `MaxChanges=1` | six Azure VM policies across both vaults classified, `candidates=0`, zero writes, job Completed |

What this proves: the text-preserving JSON layer, `Invoke-WebRequest` transport, eligibility gate,
two-phase apply, fail-closed guards and verification all work in the real Automation PowerShell 7.4
sandbox against real ARM responses. What it still does not prove: the `202` asynchronous path (every
live write completed synchronously with HTTP 200), tagged policies, V2/hourly writes, protected
items, Resource Guard/MUA, throttling, or archive movement — those remain harness-only and are listed
as required canaries in `docs/design-and-limitations.md`. (A single-item protected policy was later
written live on 2026-09-27, and that write took the `202` `Azure-AsyncOperation` path; see
[2026-09-27](#11--new-tenant-and-first-protected-policy-write-2026-09-27).)

## 1.1 — fresh replica qualification (2026-08-31)

A new Automation Account, `PowerShell74` runtime, two empty vaults, and two zero-item canary policies
were deployed in a new resource group. The published runbook was fetched back byte-for-byte and its
SHA-256 matched the public source (`2cef45acc81b04a6bbcd62582db6f974102ae98f2de79231a90907f49a7dd555`).
No tenant, subscription, resource, principal, role-assignment, or job identifier is retained here.

| Phase | Result |
|---|---|
| No-reader negative | Job Failed on the top-level vault-list permission with HTTP 403; no policy write was reachable. Because the exception occurred before the result section, there was no `SUMMARY` line. |
| Exact seed | The target policy was required to have zero protected items, then a sanitized full-document PUT changed only the `ArchivedRP` tiering block from `TierRecommended` to `DoNotTier`. |
| Audit | Completed; `WouldEnableTierRecommended`; candidates/submitted/verified = `1/0/0`; zero errors. |
| Bounded apply | Completed; `EnabledAndVerified`; candidates/submitted/verified = `1/1/1`; synchronous HTTP 200; zero failed or unknown writes. |
| Idempotent repeat | Completed; `AlreadyCompliant`; candidates/submitted/verified = `0/0/0`; zero errors. |
| Content verification | Canonical full pre/post policy SHA-256 values matched after the canary returned to `TierRecommended`; the independently calculated non-tiering SHA-256 values also matched. |
| Final access | The temporary policy-remediator assignment was removed. Only the RG-scoped discovery reader remained on the Automation identity. |
| Final platform state | Runbook Published on `PowerShell74`; both canaries `TierRecommended` with zero protected items; zero schedules, zero linked job schedules, and zero active jobs. |

This fresh run confirms that the public fixture and publishing path can be reproduced in a new
scope, including least-privilege cleanup and a retained read-only inspection state. It also corrects
the earlier documentation assumption that every runbook failure emits a structured summary.

## 1.1 — new tenant and first protected-policy write (2026-09-27)

A brand-new tenant and Azure Free Trial subscription reproduced the
[combined Policy + Automation guide](https://github.com/kevo099/azure-enterprise-policy-baseline/blob/main/docs/REPLICATE-POLICY-AUTOMATION.md)
with the unchanged helper pin `03839a29b0fff02442d88a414d7ac32851d227c7` and runbook SHA-256
`2cef45acc81b04a6bbcd62582db6f974102ae98f2de79231a90907f49a7dd555`. The Automation Account ran in
East US because Free Trial subscriptions cannot create one in Central US. The empty-canary sequence
matched earlier runs: a pre-RBAC 403 with no `SUMMARY`, reader readiness on the first job, audit
`1/0/0`, apply `1/1/1` on the first attempt, repeat `0/0/0`, and writer removal.

An optional extension then protected one private test VM with a V1 daily policy (24-month retention
horizon) and ran the runbook against that policy. Every job used exact filters, `ExpectedMatches=1`
and `MaxChanges=1`; counts are candidates/submitted/verified.

| Phase | Result |
|---|---|
| Default guard, `MaxProtectedItemsPerPolicy=0` | `SkippedProtectedItemsExceedLimit`; `0/0/0` |
| Limit raised to 1, no override | `SkippedNoConcurrencyToken`; `0/0/0`. The policy body carried no `eTag`; a separate check of the `backupPolicies` GET (`2025-08-01`) found none in the body or response headers |
| `AllowWriteWithoutETag=true` in an exclusive test window | Audit `WouldEnableTierRecommended` `1/0/0`; apply `EnabledAndVerified` `1/1/1`; repeat `AlreadyCompliant` `0/0/0` |
| Asynchronous path | The protected-policy PUT returned `202` and was followed through `Azure-AsyncOperation` to `Succeeded` before verification, the first recorded live exercise of that branch. The empty-canary apply in the same run completed synchronously with HTTP 200 |
| Invariants | Non-tiering policy properties unchanged; `protectedItemsCount` stayed 1 and the item stayed on the same policy; temporary writer removed and verified absent |
| Side effect | Azure started a `ConfigureBackup` job on the protected item right after the write |

This is the first live write to a policy that protects a real (test) VM. The item's first backup was
still running during the write, so the item held no completed recovery point yet. The run does not
cover V2/hourly, tagged or multi-item policies, mature recovery points, archive movement, restore,
or estate scale. Full record: [LIVE-TEST-2026-09-27.md](LIVE-TEST-2026-09-27.md).
