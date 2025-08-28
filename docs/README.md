# Jenkins Pipeline Documentation

## Overview
This pipeline is designed to run on a **controlled, secure agent** and provides a minimal, auditable build flow. Its primary objectives are:

- Execute builds on a restricted agent (`secure-agent`) to enforce security boundaries.  
- Supply required credentials and configuration via environment variables.  
- Perform deterministic post‑build actions: workspace cleanup, audit logging, and failure notifications.

> **Note:** The `stages` section is empty in the supplied definition. If additional build stages are required, they should be added under the `stages` block.

---

## Agent Configuration
```groovy
label 'secure-agent'
```
- **Purpose:** Guarantees that the pipeline runs only on agents tagged with `secure-agent`.  
- **Effect:** Limits execution to machines that meet the organization’s security hardening standards (e.g., hardened OS, restricted network access).

---

## Environment Variables
| Variable | Value / Source | Description |
|----------|----------------|-------------|
| `API_TOKEN` | `credentials('my-api-token')` | Securely injects an API token stored in Jenkins Credentials. Used by downstream steps that need to call external services. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | Flag to turn off any legacy or insecure functionality within the build scripts. Should be kept as `'true'` unless a controlled change is required. |

*All environment variables are automatically exported to each step of the pipeline.*

---

## Stages
> **Current state:** No stages are defined (`"stages": []`).  
> To extend the pipeline, add stage blocks such as:

```groovy
stage('Build') {
    steps {
        // build commands
    }
}
stage('Test') {
    steps {
        // test commands
    }
}
```

Each stage should include a clear purpose and the necessary steps to achieve it.

---

## Post‑Build Actions
Post actions run after the pipeline completes, regardless of success or failure.

### `always`
Executed for **every** build.

```groovy
// Clean up workspace securely
cleanWs()

// Send audit log to a secure logging service
script {
    def buildStatus = currentBuild.result ?: 'SUCCESS'
    echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
}
```

- **`cleanWs()`** – Removes all files from the workspace, ensuring no sensitive data remains on the agent.  
- **Audit script** – Logs the build number and final status (`SUCCESS` by default, otherwise the failure reason) to the Jenkins console, which can be forwarded to an external logging service.

### `failure`
Executed **only** when the build fails.

```groovy
// Notify on failure with restricted details
mail to: 'team@example.com',
     subject: "Build ${env.BUILD_NUMBER} Failed",
     body: "Check Jenkins for details: ${env.BUILD_URL}"
```

- Sends an email to the designated team with a concise failure summary and a link to the Jenkins build page.

---

## Usage Instructions

### Triggering the Pipeline
| Method | Description |
|--------|-------------|
| **Manual** | Click **Build Now** on the pipeline job page. |
| **SCM webhook** | Configure your source‑code repository to send a webhook on push/PR events to Jenkins. |
| **Parameterized trigger** | If parameters are added later, use the **Build with Parameters** UI or the Jenkins REST API. |

### Monitoring Execution
1. **Console Output** – Click the build number → **Console Output** to view real‑time logs.  
2. **Blue Ocean** – Provides a visual stage view (useful once stages are added).  
3. **Build History** – Shows status icons (green = success, red = failure).  

### Common Troubleshooting Steps
| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Build never starts | No agent with label `secure-agent` is online | Verify that at least one agent is connected and labeled correctly (`jenkins-agent` → **Configure** → **Labels**). |
| `API_TOKEN` is empty | Credential ID mismatch or missing permission | Ensure the credential `my-api-token` exists in **Jenkins > Credentials** and the pipeline job has **Read** access. |
| Post‑build email not sent | SMTP not configured or mail step fails | Check **Manage Jenkins > Configure System > E‑mail Notification** and verify the mail server settings. |
| Workspace not cleaned | `cleanWs()` step skipped or fails | Review console output for errors in the `always` block; ensure the workspace is not locked by another process. |

---

## Extending the Pipeline

1. **Add Stages** – Insert stage blocks under `stages` to perform build, test, packaging, etc.  
2. **Introduce Parallelism** – Use `parallel` inside a stage to run independent tasks concurrently.  
3. **Secure Additional Secrets** – Store any new secrets in Jenkins Credentials and reference them via `credentials('id')`.  

---

## Summary of Key Commands

| Command | Context | Role |
|---------|---------|------|
| `label 'secure-agent'` | Agent declaration | Restricts execution to approved agents. |
| `cleanWs()` | `post.always` | Securely wipes the workspace after every run. |
| `script { … }` | `post.always` | Executes Groovy code to log build status. |
| `mail to:…, subject:…, body:…` | `post.failure` | Sends a failure notification email. |
| `credentials('my-api-token')` | Environment | Retrieves a stored secret for use in the pipeline. |

--- 

*End of documentation.*