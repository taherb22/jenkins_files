# Jenkins Pipeline Documentation

## Overview

**Purpose**  
This pipeline runs on a dedicated, secure Jenkins agent (`secure-agent`) and performs an early‑stage validation of the build parameters. Its primary objective is to ensure that a branch name is supplied before any further processing occurs, preventing downstream jobs from executing with incomplete context.

**Key Objectives**
- Enforce the presence of a non‑empty `BRANCH_NAME` parameter.
- Provide a clear error message when the validation fails.
- Prepare environment variables (`API_TOKEN`, `DISABLE_INSECURE_FEATURES`) for downstream stages (not defined in this snippet).

---

## Pipeline Structure

| Section | Description |
|---------|-------------|
| **Agent** | `label 'secure-agent'` – the pipeline runs on a node tagged `secure-agent`. |
| **Environment** | Global environment variables available to all stages. |
| **Stages** | Currently a single **Initialize** stage that validates input. |
| **Post** | No post‑actions defined. |

---

## Environment Variables

| Variable | Value / Source | Purpose |
|----------|----------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Securely injects an API token stored in Jenkins credentials. Intended for API calls in later stages. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | Flags to disable any insecure features that might be enabled by default. Used by downstream scripts to enforce a hardened execution environment. |

*Note: The values are defined as strings in the pipeline DSL; Jenkins will resolve the credential reference at runtime.*

---

## Stage Details

### 1. Initialize

**Purpose**  
Validate that the required `BRANCH_NAME` parameter is provided and is not empty. This prevents the pipeline from proceeding with an undefined source branch.

**Key Activities**
- Evaluate the `params.BRANCH_NAME` value.
- Abort the build with a descriptive error if the check fails.

**Step Implementation**

```groovy
if (params.BRANCH_NAME == null || params.BRANCH_NAME.trim().isEmpty()) {
    error 'Branch name is required and cannot be empty'
}
```

- `params.BRANCH_NAME` – Jenkins‑provided build parameter (typically supplied via the UI or API).
- `trim()` – Removes leading/trailing whitespace to catch inputs that appear non‑empty but contain only spaces.
- `error` – Terminates the pipeline with the supplied message, marking the build as **FAILED**.

---

## Usage Instructions for Developers

### Triggering the Pipeline

1. **Via Jenkins UI**  
   - Navigate to the pipeline job.  
   - Click **Build with Parameters**.  
   - Provide a value for **BRANCH_NAME** (mandatory).  
   - Click **Build**.

2. **Via Jenkins REST API**  
   ```bash
   curl -X POST JENKINS_URL/job/<job-name>/buildWithParameters \
        --user <user>:<api-token> \
        --data-urlencode "BRANCH_NAME=feature/my-new-feature"
   ```

### Monitoring Execution

- **Console Output**: Click the build number in the Jenkins UI to view real‑time logs. The initialization check will appear at the top of the log.
- **Blue Ocean** (if installed): Provides a visual representation of stage progress and any failure points.

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Build fails with `Branch name is required and cannot be empty` | `BRANCH_NAME` parameter missing or blank. | Ensure the parameter is supplied and not just whitespace. |
| Build aborts before reaching later stages (if added later) | Validation step fails. | Verify the parameter value and re‑run the build. |
| Credential `my-api-token` not found | Credential ID typo or missing in Jenkins credentials store. | Add or correct the credential in **Jenkins → Credentials** and re‑run. |

---

## Extensibility Notes

- **Additional Stages**: The current pipeline only contains the `Initialize` stage. Future stages (e.g., checkout, build, test, deploy) should reference the pre‑validated `BRANCH_NAME` and the environment variables defined above.
- **Post Actions**: Consider adding `post` blocks (e.g., `always`, `success`, `failure`) to handle cleanup, notifications, or artifact archiving.

---

## Missing Information

The provided pipeline definition does not include:
- Subsequent build, test, or deployment stages.
- Post‑build actions (e.g., notifications, cleanup).
- Detailed usage of the `API_TOKEN` and `DISABLE_INSECURE_FEATURES` variables.

These sections should be added as the pipeline evolves.