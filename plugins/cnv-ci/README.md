# CNV CI Plugin

Diagnosis and analysis tools for downstream CNV Jenkins CI jobs.

## Commands

### `/cnv-ci:ci-diagnose`

Diagnose CI test failures from a Jenkins job URL by fetching JUnit results and k8s-reporter artifacts. Correlates each failure with cluster-state evidence and classifies root causes.

**Usage:**
```bash
/cnv-ci:ci-diagnose <jenkins-job-url>
```

**Prerequisites:**
- `JENKINS_USER` and `JENKINS_TOKEN` environment variables
- `curl` and `jq` on PATH
- VPN connected

## Installation

```bash
/plugin install cnv-ci@kubevirt-claude-skills
```
