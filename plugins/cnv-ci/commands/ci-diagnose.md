---
description: Diagnose CI test failures from a Jenkins job URL by fetching JUnit results and k8s-reporter artifacts
argument-hint: <jenkins-job-url>
---

## Name
kubevirt:ci-diagnose

## Synopsis
```
/kubevirt:ci-diagnose <jenkins-job-url>
```

## Description
The `kubevirt:ci-diagnose` command fetches test results and cluster-state artifacts from a downstream (CNV) Jenkins CI job and produces a root-cause diagnosis for each failed test.

Given a Jenkins build URL, it:
1. Fetches structured JUnit test results via the Jenkins API
2. Downloads k8s-reporter artifacts (events, VMIs, pods, nodes)
3. Extracts the failure message for every failed test
4. Correlates each failure with cluster-state evidence from the k8s-reporter dump
5. Classifies the root cause and outputs a diagnosis report

### Failure Classification
Each failed test is classified into one of these categories:

#### Infrastructure
- Node NotReady or under pressure (memory, disk, PID)
- Pod scheduling failures (insufficient resources, taints)
- Cluster operator degraded
- Network connectivity issues

#### Timeout
- VMI stuck in Scheduling/Pending phase
- Pod stuck in ContainerCreating/Init
- Test exceeded deadline waiting for a condition

#### Assertion
- Test expectation mismatch (wrong phase, wrong value, missing resource)
- Functional regression in KubeVirt/CNV code

#### Crash
- Pod CrashLoopBackOff or OOMKilled
- Panic in virt-handler, virt-controller, virt-launcher, or compute container
- Container exit code non-zero

## Implementation

### Phase 1: Parse and Validate
1. Parse the Jenkins URL argument to extract the build URL. Normalize it (strip trailing slash, ensure it ends with the build number).
   - Example input: `https://jenkins.example.com/job/4.99-compute-ocs-gating/262/`
   - Extracted: base URL, job path, build number
2. Validate that `JENKINS_USER` and `JENKINS_TOKEN` environment variables are set:
   ```bash
   if [ -z "$JENKINS_USER" ] || [ -z "$JENKINS_TOKEN" ]; then
     echo "Error: JENKINS_USER and JENKINS_TOKEN must be set in your environment"
     exit 1
   fi
   ```
3. Test connectivity:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" -u "$JENKINS_USER:$JENKINS_TOKEN" "${BUILD_URL}/api/json"
   ```
   If not 200, report auth or connectivity failure and stop.

### Phase 2: Fetch JUnit Test Results
1. Fetch structured test results from the Jenkins test report API:
   ```bash
   curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
     "${BUILD_URL}/testReport/api/json?tree=suites[cases[name,className,status,errorDetails,errorStackTrace,duration,stdout,stderr]]"
   ```
2. If the test report endpoint returns 404, fall back to the console log:
   ```bash
   curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" "${BUILD_URL}/consoleText"
   ```
   Parse the console log for Ginkgo failure blocks: lines matching `[FAIL]`, `Expected`, `to equal`, `Timed out`, `panic:`.
3. From the JUnit JSON, extract all test cases where `status` is `FAILED` or `REGRESSION`:
   - `name`: the full test name (includes `[test_id:XXXX]` and `[sig-compute]` labels)
   - `errorDetails`: the assertion failure message (this is the "failure reason line")
   - `errorStackTrace`: full Ginkgo stack trace
   - `duration`: how long the test ran before failing
4. Also extract summary counts: total, passed, failed, skipped.

### Phase 3: Fetch k8s-reporter Artifacts
1. List all build artifacts:
   ```bash
   curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
     "${BUILD_URL}/api/json?tree=artifacts[relativePath,fileName]"
   ```
2. Filter for k8s-reporter files. The artifact paths follow the pattern:
   ```
   test-dumps/k8s-reporter/<process-number>/<failure-count>_<type>.log
   test-dumps/k8s-reporter/suite/<failure-count>_<type>.log
   ```
3. Download the key diagnostic files to a temp directory (`$(mktemp -d)`). Priority order:
   - `*_overview.log` — pod status table (detect CrashLoop, Pending, Failed, OOMKilled)
   - `*_events.log` — Kubernetes events JSON (Warning events, FailedScheduling, FailedCreate)
   - `*_vmis.log` — VMI list JSON (phase, conditions, `kubevirt.io/created-by-test` annotation)
   - `*_vms.log` — VM list JSON (status conditions, ready state)
   - `*_nodes.log` — Node list JSON (conditions: Ready, MemoryPressure, DiskPressure, PIDPressure)
   - `*_pods.log` — Pod list JSON (detailed pod status)
   Download each with:
   ```bash
   curl -s -u "$JENKINS_USER:$JENKINS_TOKEN" \
     "${BUILD_URL}/artifact/${RELATIVE_PATH}" -o "${TEMP_DIR}/${FILENAME}"
   ```
4. Also download a sample of pod log files from `test-dumps/k8s-reporter/<N>/pods/` — specifically virt-handler, virt-controller, virt-launcher, and compute container logs. Limit to logs matching the failure count prefix.

### Phase 4: Analyze and Correlate
For each failed test from Phase 2:

1. **Extract failure message**: The `errorDetails` field is the primary "failure reason line". Extract the key assertion:
   - `Expected <X> to equal <Y>`
   - `Timed out after <duration> waiting for <condition>`
   - `panic: <message>`
   - `context deadline exceeded`

2. **Match to k8s-reporter dump**: Use the failure count number to find the corresponding dump files (e.g., failure #1 → files prefixed with `1_`). Also search `_vmis.log` for the test name in the `kubevirt.io/created-by-test` annotation to find the specific VMI.

3. **Gather cluster evidence** by scanning the matched dump:

   a. **VMI status** (from `_vmis.log`):
      - Check `.status.phase` — is it Failed, Scheduling, Pending, or Succeeded when it shouldn't be?
      - Check `.status.conditions` for error messages
      - Check `.status.reason` for failure reason
      - Note the node it was scheduled to (`.status.nodeName` or label `kubevirt.io/nodeName`)

   b. **Events** (from `_events.log`):
      - Filter for Warning events
      - Filter for events in the test's namespace (parsed from VMI metadata)
      - Look for: FailedScheduling, FailedCreate, Unhealthy, BackOff, Evicted, OOMKilling
      - Extract the `.message` field for each relevant event

   c. **Pod health** (from `_overview.log`):
      - Parse the tabular pod listing
      - Flag pods in CrashLoopBackOff, Error, OOMKilled, Pending, ImagePullBackOff
      - Flag pods with high restart counts
      - Focus on the test namespace and openshift-cnv namespace

   d. **Node conditions** (from `_nodes.log`):
      - Check each node's `.status.conditions` for:
        - `Ready != True`
        - `MemoryPressure = True`
        - `DiskPressure = True`
        - `PIDPressure = True`

   e. **Pod logs** (from `pods/` directory):
      - Scan virt-launcher and compute logs for: `panic`, `OOM`, `killed`, `error`, `fatal`
      - Scan virt-handler logs for errors related to the VMI

4. **Classify root cause**: Based on the gathered evidence, assign a category:
   - **infrastructure**: node pressure, scheduling failure, operator degraded
   - **timeout**: VMI stuck in non-terminal phase, no error events, test duration near limit
   - **assertion**: test expectation mismatch with healthy cluster state
   - **crash**: pod/container crash, OOM, panic

5. **Determine likely root cause**: Write a one-sentence diagnosis combining the failure message with the cluster evidence.

### Phase 5: Generate Report
1. Create the output directory:
   ```bash
   mkdir -p artifacts/kubevirt-ci-diagnose
   ```
2. Write the markdown report to `artifacts/kubevirt-ci-diagnose/diagnose-<job-name>-<build>-<timestamp>.md`
3. Report structure:
   ```markdown
   # CI Failure Diagnosis: <job-name> #<build>

   **URL**: <jenkins-url>
   **Date**: <build-timestamp>

   ## Summary
   - **Total**: X tests | **Passed**: Y | **Failed**: Z | **Skipped**: W
   - **Categories**: N infrastructure, M assertion, K timeout, J crash

   ## Failed Tests

   ### 1. <test-name>
   **Test ID**: CNV-XXXX
   **Duration**: Xs
   **Category**: infrastructure | timeout | assertion | crash

   **Failure message**:
   > <errorDetails — the failure reason line>

   **Cluster evidence**:
   - <bullet points of correlated cluster state>

   **Likely root cause**: <one-sentence diagnosis>

   ---

   ### 2. <next-test>
   ...

   ## Cluster Health at Failure Time
   ### Nodes
   <node conditions summary>

   ### Unhealthy Pods
   <pods not in Running/Completed state>

   ### Warning Events
   <relevant warning events>
   ```
4. Print a concise summary to stdout with the report file path.
5. Clean up the temp directory.

## Return Value
A structured diagnosis report containing:

- **Summary**: Test result counts and failure category breakdown
- **Per-test diagnosis**: For each failed test:
  - The failure reason line (from JUnit)
  - Category classification
  - Correlated cluster-state evidence
  - Likely root cause
- **Cluster health snapshot**: Node conditions, unhealthy pods, warning events at failure time

## Examples

1. **Diagnose a specific build**:
   ```
   /kubevirt:ci-diagnose https://jenkins.example.com/job/4.99-compute-ocs-gating/262/
   ```

2. **Diagnose a nested job path**:
   ```
   /kubevirt:ci-diagnose https://jenkins.example.com/job/folder/job/4.99-compute-ocs-gating/262/
   ```

3. **Diagnose without trailing slash**:
   ```
   /kubevirt:ci-diagnose https://jenkins.example.com/job/4.99-compute-ocs-gating/262
   ```

4. **Diagnose a different lane**:
   ```
   /kubevirt:ci-diagnose https://jenkins.example.com/job/4.99-network-ocs-gating/155/
   ```

5. **Diagnose latest build (if Jenkins supports "lastBuild" alias)**:
   ```
   /kubevirt:ci-diagnose https://jenkins.example.com/job/4.99-compute-ocs-gating/lastFailedBuild/
   ```

## Arguments
- `<jenkins-job-url>`: (Required) Full Jenkins build URL including job path and build number. Examples:
  - `https://jenkins.example.com/job/4.99-compute-ocs-gating/262/`
  - `https://jenkins.example.com/job/4.99-compute-ocs-gating/262`
  - `https://jenkins.example.com/job/folder/job/4.99-compute-ocs-gating/262/`

## Prerequisites
- `JENKINS_USER` environment variable set (your Jenkins username)
- `JENKINS_TOKEN` environment variable set (Jenkins API token, generated at `<jenkins-url>/user/<username>/configure`)
- `curl` available on PATH
- `jq` available on PATH (for JSON parsing)
- VPN connected (Jenkins is typically on internal network)

## Output Location
```
artifacts/kubevirt-ci-diagnose/diagnose-<job-name>-<build>-<timestamp>.md
```

## See Also
- `/kubevirt:ci-triage` - AI-powered CI failure triage with root cause analysis
- `/kubevirt:review-ci` - Review CI failures for a GitHub PR
- `/kubevirt:ci-search` - Search for specific test failures across CI jobs
- `/kubevirt:ci-lane` - Analyze failure patterns for a specific CI job lane
