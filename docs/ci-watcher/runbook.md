# CI Watcher Runbook

Jira: [ROSAENG-1442](https://redhat.atlassian.net/browse/ROSAENG-1442)

## Jobs Under Surveillance

All configured periodic jobs can be found in [Prow](https://prow.ci.openshift.org/?type=periodic&job=periodic-ci-openshift-online-rosa-e2e-main*). The jobs are organized into the following categories:

### ROSA E2E

Managed service validation using the rosa-e2e test suite:

**HCP:**
- rosa-hcp-e2e-stable-4-19
- rosa-hcp-e2e-stable-4-20
- rosa-hcp-e2e-stable-4-21
- rosa-hcp-e2e-candidate-4-22
- rosa-hcp-e2e-nightly-5-0

**Classic STS:**
- rosa-classic-sts-e2e-stable-4-19
- rosa-classic-sts-e2e-stable-4-20
- rosa-classic-sts-e2e-stable-4-21
- rosa-classic-sts-e2e-candidate-4-22

**OSD GCP:**
- osd-gcp-e2e-candidate-4-22

**Upgrade:**
- rosa-classic-sts-upgrade-4-20-to-4-21


### OCM FVT

Clusters-service API contract tests:

**ROSA HCP (staging):**
- rosa-hcp-ad-staging
- rosa-hcp-pl-staging
- rosa-hcp-shared-vpc-staging
- rosa-hcp-upgrade-staging
- rosa-hcp-y-upgrade-staging
- rosa-hcp-arm-staging
- rosa-hcp-amd64-staging
- rosa-hcp-amd64-upgrade-staging
- rosa-hcp-autonode-staging
- rosa-hcp-zero-egress-staging
- rosa-hcp-zero-egress-upgrade-staging
- rosa-hcp-adobe-staging
- hcp-e2e-staging

**ROSA HCP (integration):**
- rosa-hcp-autonode-integration
- rosa-hcp-backup-restore-integration
- rosa-hcp-bkp-cp-upgrade-integration

**ROSA Classic (staging):**
- rosa-ad-staging
- rosa-sts-ad-staging
- rosa-sts-pl-staging
- rosa-sts-shared-vpc-staging
- rosa-sts-upgrade-staging
- rosa-hcp-upgrade-staging
- ocm-resources-staging
- osd-rh-aws-staging

**ROSA Classic (integration):**
- rosa-sts-ad-integration

**OSD GCP (staging):**
- osd-ccs-gcp-ad-staging
- osd-ccs-gcp-marketplace-staging
- osd-gcp-non-cross-proj-wif-staging

### OCP Conformance

OCP conformance tests (openshift-tests) run against ROSA clusters on each OCP nightly. These jobs live in the [openshift/release](https://github.com/openshift/release) repo and can be found in [Prow](https://prow.ci.openshift.org/?job=periodic-ci-openshift-release-main-nightly*rosa*).

**ROSA HCP:**
- nightly-4.19-e2e-rosa-hcp-ovn
- nightly-4.20-e2e-rosa-hcp-ovn
- nightly-4.21-e2e-rosa-hcp-ovn
- nightly-4.22-e2e-rosa-hcp-ovn
- nightly-5.0-e2e-rosa-hcp-ovn

**ROSA Classic STS:**
- nightly-4.18-e2e-rosa-sts-ovn
- nightly-4.19-e2e-rosa-sts-ovn
- nightly-4.20-e2e-rosa-sts-ovn
- nightly-4.21-e2e-rosa-sts-ovn
- nightly-4.22-e2e-rosa-sts-ovn

## Daily Triage Procedure

### Step 1: Run /ci-triage

```
/ci-triage
```

This spawns a team of 4 background agents:

| Agent | Role |
|-------|------|
| status-checker | Polls all tracked jobs, reports state changes |
| log-analyzer | Deep-dives into specific failures on demand |
| fix-proposer | Creates fix PRs for test bugs and config drift |
| jira-implementer | Creates Jira stories, shepherds open PRs |

Wait for agent reports to complete before proceeding.

### Step 2: Check MC/SC Health

Before triaging individual test results, check the health of the CI management clusters and service clusters. Degraded infrastructure produces cascading failures that look like individual test bugs but are actually systemic.

Check for:
- **Error cluster ratio**: High proportion of HCPs in error state on the CI MC
- **Stuck deletions**: Clusters with deletionTimestamp set but still present (orphaned resources)
- **Developer HCPs on CI MCs**: Personal test clusters consuming capacity on CI infrastructure
- **Certificate issues**: STS AssumeRole failures, OIDC validation failures

If a MC is degraded (high error ratio, many stuck clusters), flag it immediately in `#wg-rosa-ci-enhancement`. Individual test failures on a degraded MC should be attributed to the infrastructure issue, not the tests.

### Step 3: Review Against Sippy

Open the [Sippy rosa-stage dashboard](https://sippy.dptools.openshift.org/sippy-ng/release/rosa-stage) and cross-reference with `/ci-triage` findings. Look for:
- Jobs showing declining pass rates over multiple days
- New failures that appeared overnight
- Jobs with no recent data (may indicate a config or infra problem)

### Step 4: Check Component Readiness

Open the ROSA component readiness views in Sippy for the latest Y-stream versions:

| OCP Version | Sippy View |
|-------------|-----------|
| 4.22 | [4.22-rosa](https://sippy-auth.dptools.openshift.org/sippy-ng/component_readiness/main?view=4.22-rosa) |
| 5.0 | [5.0-rosa](https://sippy-auth.dptools.openshift.org/sippy-ng/component_readiness/main?view=5.0-rosa) |

Additional views may be added as new OCP versions enter the pipeline. Check [Sippy component readiness](https://sippy-auth.dptools.openshift.org) for the latest.

Look for:
- **Red components**: Regressions detected — these need immediate attention
- **Yellow components**: Flapping or borderline — monitor over the next 1-2 days
- **Green components**: Healthy — no action needed

Component readiness may surface regressions that individual job pass/fail misses (e.g., a test that started failing in one of many suites). Conversely, a red job may not show up in component readiness if the failing tests are not in the monitored set. Use both signals together.

Regressions showing in component readiness views are visible to TRT and can block OCP release promotion. File and track them immediately:
- File Jira under [ROSAENG-391](https://redhat.atlassian.net/browse/ROSAENG-391)
- Include the component readiness view link and screenshot if applicable
- Route to the owning team per the [classification matrix](escalation-paths.md#classification-matrix)
- Follow the [conformance failure response targets](escalation-paths.md#response-targets)

### Step 5: Investigate Unclassified Failures

For any failures `/ci-triage` could not classify:
1. Navigate to the job in [Prow](https://prow.ci.openshift.org/)
2. Pull the build logs from GCS: `storage.googleapis.com/test-platform-results/logs/<job-name>/<build-id>/`
3. Key files to check: `finished.json`, `build-log.txt`, step artifacts
4. Classify the failure using the [classification matrix](escalation-paths.md#classification-matrix)

### Step 6: Take Action

- Review and approve Jira stories proposed by `/ci-triage`
- Review and merge fix PRs (request a reviewer — do not self-lgtm)
- File Jira under [ROSAENG-391](https://redhat.atlassian.net/browse/ROSAENG-391) for anything the AI missed

### Step 7: Review Daily Status Report

An automated [CI status report](../../scripts/ci-status-report.sh) is posted daily to `#wg-rosa-ci-enhancement`. Use this report as your starting point to track job health across all monitored jobs. The report reads the canonical [job list](../../configs/ci-status-jobs.yaml) and queries Prow GCS for the latest results.

Review the report for:
- Jobs that have turned red since the last report
- Persistent failures that need Jira stories or escalation
- Jobs with no recent data that may need investigation

## Weekly Handover Procedure

### Friday: Write Handover

Create a handover document following this template:

```markdown
# ROSA CI Handover - YYYY-MM-DD

## Current Status (N jobs tracked)

### Healthy
- list of passing jobs

### Persistent Failures (diagnosed, fixes in progress)
- job name: description, Jira link, PR link

### No Data
- jobs with no recent builds

## Open PRs Needing Review
- Grouped by repository

## Monday Action Items
1. Concrete next steps for incoming watcher
```

### Friday: Post and Tag

1. Post the handover to `#wg-rosa-ci-enhancement`
2. Tag the next watcher by name
3. Post the weekly CI status summary

### Monday: Incoming Watcher

1. Read the handover document from the outgoing watcher
2. Run `/ci-triage` to verify continuity of tracked issues
3. Check that any "in progress" fixes from last week have merged
4. Post an opening status to `#wg-rosa-ci-enhancement`

## Common Scenarios

### Conformance Job Turns Red

1. Check if `/ci-triage` already caught and classified the failure
2. If it's an OCP regression (test passes on prior nightly, fails on new one), file an upstream OCPBUGS-\* bug and add a temporary skip with a link to the bug
3. If it's a ROSA-side issue (cluster provisioning, STS policy, config drift), route to the appropriate ROSA team
4. Ack within 4 business hours in the reporting channel
5. If unresolved after 24 hours, escalate via WebRCA and bring in the relevant teams

### MC/SC Appears Degraded

1. Flag immediately in `#wg-rosa-ci-enhancement`
2. Circuit-break further test triage until MC is healthy
3. Attribute individual test failures on a degraded MC to the infrastructure issue
4. Related: [OCM-23872](https://redhat.atlassian.net/browse/OCM-23872) tracks making dev SCs/MCs healthy to reduce CI noise

### Cluster not ready due to readiness check failure

1. Flag immediately in `#wg-rosa-ci-enhancement`
2. Identify the component which is blocking the cluster readiness check
3. Fix the problem to unblock the tests if issue identified
4. Find the owner team of the component as high priority if the fix is not obvious

### All Jobs Green

Post "All Clear" to `#wg-rosa-ci-enhancement`. This is still worth posting — it confirms you checked.
