# Jenkins Pipeline Documentation

## Overview

**Pipeline Name:** *Unnamed (provided JSON)*  
**Agent:** `label 'secure-agent'`

This pipeline is designed to perform an early validation step before any further build, test, or deployment actions. Its primary objective is to ensure that a branch name is supplied when the pipeline is triggered, preventing downstream failures caused by missing or empty branch identifiers.

---

## Table of Contents

1. [Purpose & Objectives](#purpose--objectives)  
2. [Pipeline Structure](#pipeline-structure)  
   - [Agent Configuration](#agent-configuration)  
   - [Environment Variables](#environment-variables)  
   - [Stages](#stages)  
3. [Stage Details](#stage-details)  
   - [Initialize](#initialize-stage)  
4. [Running the Pipeline](#running-the-pipeline)  
5. [Monitoring & Logs](#monitoring--logs)  
6. [Troubleshooting](#troubleshooting)  
7. [Environment Variable Reference](#environment-variable-reference)  
8. [Notes on Missing Sections](#notes-on-missing-sections)  

---

## Purpose & Objectives

- **Validate Input:** Ensure that the `BRANCH_NAME` parameter is provided and not empty.
- **Fail Fast:** Abort the pipeline early if the validation fails, saving compute resources and providing immediate feedback to developers.
- **Secure Execution:** Run on a dedicated agent labeled `secure-agent` and use secured credentials for any downstream API interactions (though not used in the current stage).

---

## Pipeline Structure

### Agent Configuration
```groovy
agent { label 'secure-agent' }
```
- The pipeline runs on any Jenkins node that carries the label **secure-agent**.  
- This isolates the execution to a controlled environment, typically hardened for security‑sensitive tasks.

### Environment Variables
| Variable | Value | Description |
|----------|-------|-------------|
| `API_TOKEN` | `credentials('my-api-token')` | Retrieves a secret API token from Jenkins Credentials store. Intended for downstream API calls. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | Flag to disable any insecure features that might be enabled elsewhere in the pipeline. Currently set to a static string `'true'`. |

> **Note:** The values are defined as Groovy strings; the actual token is resolved at runtime by Jenkins.

### Stages
| Stage | When Condition | Purpose |
|-------|----------------|---------|
| **Initialize** | *None* (always runs) | Validate that `BRANCH_NAME` is supplied and non‑empty. |

---

## Stage Details

### Initialize Stage
**Purpose:** Guard against missing branch information.

**Key Activities:**
1. **Parameter Check** – Evaluates the `BRANCH_NAME` parameter.
2. **Error Handling** – Calls `error` to abort the build with a clear message if validation fails.

**Groovy Script Executed:**
```groovy
if (params.BRANCH_NAME == null || params.BRANCH_NAME.trim().isEmpty()) {
    error 'Branch name is required and cannot be empty'
}
```

- `params.BRANCH_NAME` – Expected to be supplied either via a **Multibranch Pipeline** or as a **String Parameter** when manually triggering the job.
- `trim()` – Removes surrounding whitespace to catch cases where the parameter is present but blank.
- `error` – Marks the build as **FAILED** and stops further stage execution.

---

## Running the Pipeline

1. **Trigger Manually**  
   - Navigate to the Jenkins job page.  
   - Click **Build with Parameters**.  
   - Provide a value for **BRANCH_NAME** (e.g., `feature/login`).  
   - Click **Build**.

2. **Automatic Trigger (Multibranch)**  
   - When a new branch is created in the SCM repository, Jenkins automatically scans and creates a job for that branch.  
   - The `BRANCH_NAME` parameter is populated by Jenkins; no manual input is required.

3. **API Trigger**  
   - Use Jenkins REST API:  
     ```bash
     curl -X POST JENKINS_URL/job/your-pipeline/buildWithParameters \
          --user USER:API_TOKEN \
          --data-urlencode "BRANCH_NAME=release/v1.2"
     ```

---

## Monitoring & Logs

- **Console Output:** Accessible from the build’s **Console Output** link. The validation message will appear early in the log.
- **Blue Ocean:** Provides a visual representation of stage execution and any failure points.
- **Build Status:** The pipeline will be marked **FAILED** if the branch validation fails; otherwise, it proceeds to subsequent stages (if any are added later).

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| `Branch name is required and cannot be empty` | `BRANCH_NAME` not supplied or blank. | Ensure the parameter is provided and contains a non‑empty string. |
| Build hangs on the Initialize stage | Agent `secure-agent` unavailable or offline. | Verify that at least one Jenkins node with the label `secure-agent` is online and has sufficient executors. |
| Credential lookup fails (`my-api-token` not found) | Credential ID missing or mis‑typed. | Add a **Secret Text** credential with ID `my-api-token` in **Jenkins > Credentials**. |
| Environment variable `DISABLE_INSECURE_FEATURES` not respected | Downstream steps ignore the flag. | Ensure any downstream scripts or tools read the variable (e.g., `if [ "$DISABLE_INSECURE_FEATURES" = "true" ]; then …`). |

---

## Environment Variable Reference

| Variable | Source | Example Value | Usage |
|----------|--------|---------------|-------|
| `API_TOKEN` | Jenkins Credentials (`my-api-token`) | *resolved at runtime* | Securely pass an API token to downstream HTTP calls. |
| `DISABLE_INSECURE_FEATURES` | Static string `'true'` | `'true'` | Toggle to disable features that are considered insecure; downstream scripts should check this flag. |
| `BRANCH_NAME` | Build parameter (provided by user or SCM) | `feature/add-login` | Identifies the source branch for the build. Required for pipeline execution. |

---

## Notes on Missing Sections

- **`post` Section:** The provided JSON contains an empty `post` block. No post‑build actions (e.g., cleanup, notifications) are defined. Consider adding `always`, `success`, `failure`, or `unstable` handlers as needed.
- **Additional Stages:** Only the **Initialize** stage is defined. Future stages (e.g., Build, Test, Deploy) should be added to extend the pipeline’s functionality.
- **`when` Conditions:** The `when` array for the stage is empty, meaning the stage always runs. Conditional execution can be added later if required.