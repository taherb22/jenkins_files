# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build flow. Its primary objectives are:

- **Enforce execution on a trusted node** to reduce exposure to insecure environments.  
- **Expose required credentials** (`API_TOKEN`) and enforce security flags (`DISABLE_INSECURE_FEATURES`).  
- **Perform deterministic post‑build actions** such as workspace cleanup, audit logging, and failure notifications.

> **Note:** The `stages` section is empty in the supplied definition. Consequently, the pipeline currently performs no build or test steps. Add stages as needed for your project’s CI/CD workflow.

---

## Agent Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| **Label** | `secure-agent` | Restricts the pipeline to run only on agents that carry the `secure-agent` label. This ensures a known, hardened execution environment. |

```groovy
agent {
    label 'secure-agent'
}
```

---

## Environment Variables

| Variable | Source | Default / Value | Purpose |
|----------|--------|----------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | (masked) | Securely injects an API token from Jenkins Credentials Store. Used by downstream steps that need to call external services. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | `true` | Global flag to disable any legacy or insecure functionality within the pipeline or invoked scripts. |

```groovy
environment {
    API_TOKEN = credentials('my-api-token')
    DISABLE_INSECURE_FEATURES = 'true'
}
```

---

## Stages

> **Current State:** No stages are defined (`"stages": []`).  
> **Action Required:** Populate the `stages` array with the necessary build, test, packaging, or deployment steps for your project.

*Example placeholder:*

```groovy
stages {
    stage('Build') {
        steps {
            // build commands here
        }
    }
    // Additional stages …
}
```

---

## Post‑Build Actions

The pipeline defines two post‑conditions: **always** and **failure**.

### 1. `always`

Executed after every run, regardless of success or failure.

| Step | Command | Explanation |
|------|---------|-------------|
| **Workspace Cleanup** | `cleanWs()` | Securely deletes the workspace to prevent residue data from persisting on the agent. |
| **Audit Log** | ```groovy\nscript {\n    def buildStatus = currentBuild.result ?: 'SUCCESS'\n    echo \"Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}\"\n}\n``` | Emits a concise log entry containing the build number and final status. This message can be forwarded to external logging services via Jenkins system log configuration. |

### 2. `failure`

Executed **only** when the pipeline ends with a failure.

| Step | Command | Explanation |
|------|---------|-------------|
| **Failure Notification** | ```groovy\nmail to: 'team@example.com',\n     subject: \"Build ${env.BUILD_NUMBER} Failed\",\n     body: \"Check Jenkins for details: ${env.BUILD_URL}\" \n``` | Sends an email to the designated team with a minimal payload (build number and URL) to avoid leaking sensitive details. |

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Command / UI Action |
|--------|---------------------|
| **Manual** | Click **Build Now** on the pipeline’s Jenkins job page. |
| **SCM Hook** | Configure your repository webhook (e.g., GitHub, GitLab) to trigger the job on push/PR events. |
| **Parameterized Build** | If you later add parameters, use the **Build with Parameters** UI or the Jenkins REST API (`POST /job/<job-name>/buildWithParameters`). |

### Monitoring Execution

1. **Console Output** – Access the live console log from the build’s page to view step‑by‑step output.  
2. **Blue Ocean** – Use the Blue Ocean UI for a visual representation of stages (once stages are added).  
3. **Build Summary** – The `always` post block logs a concise status line (`Build #X completed with status: Y`).  

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Pipeline never starts** | Agent with label `secure-agent` unavailable. | Verify that at least one online agent carries the `secure-agent` label. |
| **Missing `API_TOKEN`** | Credential ID typo or missing credential. | Ensure a credential named `my-api-token` exists in **Jenkins > Credentials** and is accessible to the pipeline’s folder/job. |
| **Workspace not cleaned** | `cleanWs()` step skipped due to early abort. | Check for `catchError` or `timeout` blocks that may bypass `post` sections; adjust as needed. |
| **Failure email not sent** | Mail plugin misconfigured or SMTP unreachable. | Verify Jenkins **Configure System → E‑mail Notification** settings and test with a simple `mail` step in a sandbox job. |

---

## Extending the Pipeline

When adding stages, follow these best practices:

1. **Label each stage clearly** (e.g., `Checkout`, `Build`, `Test`, `Deploy`).  
2. **Wrap sensitive commands** in `withCredentials` blocks if additional secrets are required.  
3. **Use `try / catch`** to capture errors and set `currentBuild.result` appropriately, ensuring the `post` sections behave as expected.  
4. **Maintain the `always` cleanup** to keep the agent clean after every run.

*Sample stage skeleton:*

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.  
- **Environment:** Secure token (`API_TOKEN`) and a hardening flag (`DISABLE_INSECURE_FEATURES`).  
- **Post‑Build:** Guarantees workspace cleanup, audit logging, and failure notifications.  
- **Current Gap:** No functional stages are defined; developers should add the required build/test steps.  

By adhering to the guidelines above, teams can safely integrate this pipeline into their CI/CD process while maintaining a strong security posture.