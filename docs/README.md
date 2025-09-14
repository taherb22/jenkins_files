# Jenkins Pipeline Documentation

## Overview

**Pipeline Name:** *Unnamed (single‑stage pipeline)*  
**Agent:** `label 'trusted-builder'` – runs on any Jenkins node labeled **trusted-builder**.

**Purpose:**  
Automates the build and test process for the project by executing a predefined test script. The pipeline is intended to be a lightweight, fast‑feedback mechanism that validates code changes before they are merged.

---

## Pipeline Summary

| Item | Description |
|------|-------------|
| **Agent** | Executes on a trusted builder node (`trusted-builder`). |
| **Stages** | One stage – **Build and Test**. |
| **Environment Variables** | None defined in the pipeline (see section *Environment Variables*). |
| **Post Actions** | Not defined (pipeline ends after the stage). |

---

## Stage Details

### 1. Build and Test

| Attribute | Value |
|-----------|-------|
| **When** | No conditional execution defined – the stage always runs. |
| **Purpose** | Compile (if needed) and run the project's test suite to ensure code quality and functional correctness. |
| **Key Activities** | 1. Log a message indicating the start of the process.<br>2. Execute the test script `run_tests.sh`. |

#### Step‑by‑Step Breakdown

| Step | Command | Explanation |
|------|---------|-------------|
| **Step 1** | `echo 'Running build and test procedures...'` | Prints a clear, human‑readable message to the Jenkins console log, helping operators identify the stage’s start. |
| **Step 2** | `sh './run_tests.sh'` | Executes the shell script `run_tests.sh` located at the workspace root. This script should contain all build and test commands (e.g., compilation, unit/integration tests, linting). The `sh` step captures the script’s exit code; a non‑zero exit will fail the stage and abort the pipeline. |

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Description |
|--------|-------------|
| **Manual** | Open the Jenkins job page and click **Build Now**. |
| **SCM Trigger** | If configured in the Jenkinsfile (not shown), commits or pull‑request events can automatically start the pipeline. |
| **API** | Use the Jenkins REST API: `POST JENKINS_URL/job/<job-name>/build` (requires appropriate credentials). |

### Monitoring Execution

1. **Console Output** – Click the build number, then **Console Output** to view real‑time logs. Look for the “Running build and test procedures...” message to confirm the stage started.
2. **Stage View** – The **Stage View** plugin (if installed) visualizes stage progress and duration.
3. **Artifacts & Test Reports** – If `run_tests.sh` publishes JUnit XML or other artifacts, they will appear under **Test Result** or **Artifacts** sections (requires additional pipeline steps not present here).

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Pipeline fails immediately** | `run_tests.sh` missing or not executable. | Verify the script exists in the repository root and has execute permission (`chmod +x run_tests.sh`). |
| **No output from tests** | Script runs but suppresses output. | Modify `run_tests.sh` to echo progress or use `set -x` for shell debugging. |
| **Stage hangs** | Script waiting for input or long‑running process. | Ensure the script runs non‑interactively; add timeouts or use `timeout` command if needed. |
| **Unexpected exit code** | Tests failing or script returning non‑zero status. | Review the script’s exit logic; fix failing tests or adjust the script to return appropriate codes. |

---

## Environment Variables

The provided pipeline does **not** declare any environment variables. If your project requires configuration (e.g., credentials, paths, flags), consider adding them under the `environment` block, for example:

```groovy
environment {
    JAVA_HOME = '/usr/lib/jvm/java-11-openjdk'
    TEST_ENV  = 'staging'
}
```

Document any added variables here, describing their purpose and expected values.

---

## Extending the Pipeline (Optional)

While the current pipeline is minimal, you may want to add:

* **Additional Stages** – e.g., *Static Analysis*, *Package*, *Deploy*.
* **Post Actions** – e.g., `always { cleanWs() }` to clean the workspace, or `failure { mail to: 'dev-team@example.com', subject: 'Build Failed', body: 'Check console log.' }`.
* **Parallel Execution** – Run tests in parallel across multiple agents for faster feedback.

These enhancements follow the same declarative syntax demonstrated above.