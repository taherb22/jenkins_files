# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build process. Its primary objectives are:

- **Enforce execution on a trusted node** using a specific agent label.
- **Securely manage credentials** (API token) and enforce security‑related environment flags.
- **Perform consistent cleanup** and audit logging after every run.
- **Notify the team** when a build fails, while limiting the amount of exposed information.

> **Note:** The `stages` section is empty in the supplied definition. No build, test, or deployment steps are currently defined. The documentation below reflects the existing configuration and highlights where additional stages should be added.

---

## Agent Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| **Label** | `secure-agent` | Restricts the pipeline to run only on agents that carry the `secure-agent` label. This ensures a controlled environment with the required security hardening. |

```groovy
agent {
    label 'secure-agent'
}
```

---

## Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Retrieves a secret API token from Jenkins Credentials Store. The token is injected into the build environment for any steps that need to call external services. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | A flag used by downstream scripts or tools to disable any functionality that is considered insecure. It should be respected by all scripts executed in the pipeline. |

*All environment variables are defined at the top‑level `environment` block, making them available to every stage and post step.*

---

## Stages

> **Current status:** No stages are defined (`"stages": []`).  
> To implement actual build logic, add one or more stage blocks under the `stages` array, for example:

```groovy
stages {
    stage('Checkout') {
        steps {
            // git checkout commands
        }
    }
    stage('Build') {
        steps {
            // compilation commands
        }
    }
    // Additional stages …
}
```

Each stage should include a clear purpose, required steps, and any relevant error handling.

---

## Post‑Build Actions

The pipeline defines two post conditions: **always** and **failure**.

### `always`

Executed after every build, regardless of outcome.

| Step | Description |
|------|-------------|
| `cleanWs()` | Securely deletes the workspace to prevent credential leakage or leftover artifacts. |
| Audit logging script | Captures the final build status (`SUCCESS` by default) and prints a concise message to the console. This information can be redirected to a secure logging service if desired. |

```groovy
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
}
```

### `failure`

Executed **only** when the build result is `FAILURE`.

| Step | Description |
|------|-------------|
| Email notification | Sends a concise failure email to `team@example.com`. The email includes the build number and a link to the Jenkins build page for further investigation. Sensitive details are omitted to comply with security policies. |

```groovy
post {
    failure {
        // Notify on failure with restricted details
        mail to: 'team@example.com',
             subject: "Build ${env.BUILD_NUMBER} Failed",
             body: "Check Jenkins for details: ${env.BUILD_URL}"
    }
}
```

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Command / Action |
|--------|------------------|
| **Manual** | Click **Build Now** on the pipeline’s Jenkins job page. |
| **SCM Change** | Configure a webhook (e.g., GitHub, GitLab) to trigger the job on push/PR events. |
| **Parameterized Build** | If parameters are added later, use the **Build with Parameters** UI or the Jenkins REST API. |

### Monitoring Execution

1. **Console Output** – Click the build number to view real‑time logs. Look for the audit line: `Build <#> completed with status: <STATUS>`.
2. **Workspace** – The workspace is automatically cleaned after each run (`cleanWs()`), so no manual inspection is required.
3. **Email Alerts** – On failure, an email is sent to `team@example.com`. Verify receipt if a build fails unexpectedly.

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Build never starts** | Agent with label `secure-agent` is offline or missing. | Verify that at least one agent is online and labeled `secure-agent`. |
| **Missing `API_TOKEN`** | Credential ID `my-api-token` does not exist or is mis‑typed. | Go to **Jenkins → Credentials**, ensure the ID matches, and that the token has appropriate permissions. |
| **Email not sent on failure** | SMTP configuration is incorrect or the `mail` step is disabled. | Check **Manage Jenkins → Configure System → E‑mail Notification** and confirm the SMTP server settings. |
| **Workspace not cleaned** | `cleanWs()` step fails (e.g., permission issue). | Review the console log for errors; ensure the agent user has permission to delete the workspace directory. |

---

## Extending the Pipeline

When adding new stages, follow these best practices:

1. **Label each stage clearly** (e.g., `Checkout`, `Build`, `Test`, `Deploy`).
2. **Wrap sensitive commands** in `withCredentials` blocks if additional secrets are required.
3. **Maintain security posture** by respecting `DISABLE_INSECURE_FEATURES` in any scripts or tools invoked.
4. **Add appropriate post actions** (e.g., archiving artifacts, publishing test reports) inside the `post` block or within individual stages as needed.

```groovy
stage('Test') {
    steps {
        // Example: run unit tests
        sh 'npm test'
    }
    post {
        success {
            junit 'reports/**/*.xml'
        }
        failure {
            // Additional failure handling if required
        }
    }
}
```

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.
- **Environment:** Secure API token and a flag to disable insecure features.
- **Post‑Build:** Workspace cleanup, audit logging (always), and failure email notification.
- **Current Gaps:** No functional stages are defined; developers should add build, test, and deployment stages as needed.

This documentation provides a concise reference for developers to understand, run, and extend the pipeline while maintaining the security constraints already in place.