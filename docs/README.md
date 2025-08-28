# Jenkins Pipeline Documentation

## Table of Contents
1. [Purpose & Objectives](#purpose--objectives)  
2. [Pipeline Overview](#pipeline-overview)  
   - [Agent Configuration](#agent-configuration)  
   - [Stages](#stages)  
   - [Environment Variables](#environment-variables)  
   - [Post‑Build Actions](#post-build-actions)  
3. [Developer Usage Guide](#developer-usage-guide)  
   - [Triggering the Pipeline](#triggering-the-pipeline)  
   - [Monitoring Execution](#monitoring-execution)  
   - [Common Troubleshooting Steps](#common-troubleshooting-steps)  
4. [Appendix](#appendix)  
   - [Full Pipeline Snippet (for reference)](#full-pipeline-snippet)  

---

## Purpose & Objectives
This pipeline is designed to run **securely** on a controlled Jenkins agent, enforce a minimal set of environment constraints, and guarantee clean‑up and audit logging after every execution. Its primary objectives are:

- **Isolation:** Execute only on agents labeled `secure-agent`.
- **Security:** Use credential‑bound tokens and disable insecure features.
- **Auditing:** Emit a concise build summary and forward it to a secure logging service.
- **Failure Notification:** Alert the responsible team with limited, non‑sensitive details when a build fails.

---

## Pipeline Overview

### Agent Configuration
```groovy
label 'secure-agent'
```
- **What it does:** Restricts the pipeline to run on Jenkins agents that carry the `secure-agent` label.  
- **Why it matters:** Guarantees that the build runs in a hardened environment with pre‑approved tooling and network access.

### Stages
> **Note:** The supplied pipeline definition does not contain any stages (`"stages": []`).  
If stages are required (e.g., checkout, build, test, deploy), they should be added under the `stages` block following standard Declarative Pipeline syntax.

### Environment Variables
| Variable | Definition | Purpose |
|----------|------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Retrieves a secret API token from Jenkins Credentials Store (ID: `my-api-token`). The token is injected as a masked environment variable for use by downstream steps (e.g., API calls). |
| `DISABLE_INSECURE_FEATURES` | `'true'` | A flag that downstream scripts can read to disable any legacy or insecure functionality. Keeping it set to `true` enforces a security‑first posture. |

> **Tip:** All environment variables are automatically exported to each step in the pipeline.

### Post‑Build Actions
#### `always`
Executed **regardless** of build outcome.

```groovy
// Clean up workspace securely
cleanWs()

// Send audit log to a secure logging service
script {
    def buildStatus = currentBuild.result ?: 'SUCCESS'
    echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
}
```
- **`cleanWs()`** – Removes all files from the workspace, ensuring no artefacts leak between builds.
- **Audit script** – Captures the final build status (`SUCCESS` by default) and prints a concise log line. This line can be forwarded to an external logging service via Jenkins log aggregation.

#### `failure`
Executed **only** when the build ends in a failure state.

```groovy
// Notify on failure with restricted details
mail to: 'team@example.com',
     subject: "Build ${env.BUILD_NUMBER} Failed",
     body: "Check Jenkins for details: ${env.BUILD_URL}"
```
- **`mail` step** – Sends an email to the designated distribution list with a minimal payload (build number and URL) to avoid leaking sensitive data.

---

## Developer Usage Guide

### Triggering the Pipeline
| Method | Description |
|--------|-------------|
| **Manual** | Open the Jenkins job UI, click **Build Now**. |
| **SCM Webhook** | Configure your source‑code repository (GitHub, GitLab, Bitbucket, etc.) to send a webhook to Jenkins. The pipeline will start on each push/PR according to the job’s *Branch Specifier*. |
| **Parameterized Trigger** | If the job is set up with parameters (not shown in the current definition), use the **Build with Parameters** UI or the Jenkins REST API (`POST /job/<name>/buildWithParameters`). |

### Monitoring Execution
1. **Console Output** – Click the build number in Jenkins UI → **Console Output** to view real‑time logs, including the audit echo line.
2. **Blue Ocean** – If installed, use Blue Ocean for a visual stage view (will show an empty stage list until stages are added).
3. **Build History** – The Jenkins dashboard provides quick status icons (blue = success, red = failure).

### Common Troubleshooting Steps
| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Build fails before any stage runs | Agent not available or label mismatch | Verify that at least one agent is online with the `secure-agent` label (`Manage Jenkins → Nodes`). |
| `API_TOKEN` is masked as `<***>` and steps cannot authenticate | Credential ID typo or missing permission | Ensure the credential with ID `my-api-token` exists and the pipeline’s job has **Read** permission on it. |
| Email not sent on failure | SMTP not configured or mail step mis‑typed | Check **Manage Jenkins → Configure System → E‑mail Notification** and confirm the `mail` step syntax. |
| Workspace not cleaned | `cleanWs()` plugin missing | Install the **Workspace Cleanup Plugin** from the Jenkins plugin manager. |
| No logs appear in external logging service | Audit echo not captured by external system | Verify that the external log collector is subscribed to Jenkins logs (e.g., via Logstash, Splunk, or CloudWatch). |

---

## Appendix

### Full Pipeline Snippet (Reference)

```groovy
pipeline {
    agent {
        // Use a specific, controlled agent label to limit where the pipeline runs
        label 'secure-agent'
    }

    environment {
        API_TOKEN = credentials('my-api-token')
        DISABLE_INSECURE_FEATURES = 'true'
    }

    stages {
        // No stages defined – add your build steps here
    }

    post {
        always {
            // Clean up workspace securely
            cleanWs()

            // Send audit log to a secure logging service
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
            }
        }

        failure {
            // Notify on failure with restricted details
            mail to: 'team@example.com',
                 subject: "Build ${env.BUILD_NUMBER} Failed",
                 body: "Check Jenkins for details: ${env.BUILD_URL}"
        }
    }
}
```

*Add stages as needed to implement checkout, build, test, and deployment logic while preserving the security posture defined above.*