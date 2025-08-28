# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build flow. Its primary objectives are:

- **Enforce execution on a trusted node** using a dedicated label.
- **Securely manage sensitive credentials** (API token) via Jenkins Credentials.
- **Guarantee workspace cleanup** and audit logging after every run.
- **Notify the team on failures** while limiting exposed information.

> **Note:** The `stages` section is empty in the supplied definition. No build, test, or deployment steps are currently defined. Add stages as needed for your project’s workflow.

---

## Agent Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| `label` | `secure-agent` | Restricts the pipeline to run only on agents that carry the `secure-agent` label. This helps isolate the build environment and enforce security policies. |

```groovy
agent {
    label 'secure-agent'
}
```

---

## Environment Variables

| Variable | Source | Default / Value | Purpose |
|----------|--------|----------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | (retrieved at runtime) | Securely injects an API token stored in Jenkins Credentials. Use `${env.API_TOKEN}` in scripts that need to authenticate against external services. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | `true` | A flag that can be read by downstream scripts to disable any functionality deemed insecure. |

```groovy
environment {
    API_TOKEN = credentials('my-api-token')
    DISABLE_INSECURE_FEATURES = 'true'
}
```

---

## Stages

> **Current status:** No stages are defined (`"stages": []`).  
> To implement actual work (e.g., checkout, build, test, deploy), add stage blocks under the `stages` section.

*Example placeholder:*

```groovy
stages {
    stage('Example') {
        steps {
            echo 'Add your build steps here.'
        }
    }
}
```

---

## Post‑Build Actions

The `post` block runs after the pipeline completes, regardless of success or failure.

### `always`

Executed after **every** run.

1. **Workspace Cleanup**  
   ```groovy
   cleanWs()
   ```
   - Removes all files from the workspace to prevent data leakage between builds.

2. **Audit Log Emission**  
   ```groovy
   script {
       def buildStatus = currentBuild.result ?: 'SUCCESS'
       echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
   }
   ```
   - Captures the final build status (`SUCCESS`, `FAILURE`, etc.) and prints a concise audit line.
   - This output can be forwarded to a centralized logging service via Jenkins log aggregation.

### `failure`

Executed **only** when the pipeline ends with a failure.

1. **Restricted Failure Notification**  
   ```groovy
   mail to: 'team@example.com',
        subject: "Build ${env.BUILD_NUMBER} Failed",
        body: "Check Jenkins for details: ${env.BUILD_URL}"
   ```
   - Sends an email to the designated team address.
   - The message contains only the build number and a link to the Jenkins build page, avoiding exposure of sensitive logs.

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Description |
|--------|-------------|
| **Manual start** | Click **Build Now** on the pipeline’s Jenkins job page. |
| **SCM webhook** | Configure your source‑code repository (GitHub, GitLab, etc.) to send a webhook on push/PR events to the Jenkins job URL. |
| **Parameterized trigger** | If you later add parameters, use the **Build with Parameters** UI or the Jenkins REST API (`/job/<job-name>/buildWithParameters`). |

### Monitoring Execution

1. **Console Output** – Real‑time logs are available via the **Console Output** link on the build page.
2. **Blue Ocean** – For a visual pipeline view, open the job in Blue Ocean.
3. **Build Summary** – The audit line printed in the `always` post step (`Build # – status`) appears at the end of the console log and can be indexed by external log aggregators.

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Pipeline never starts** | No agent with label `secure-agent` is online. | Verify that at least one Jenkins node is labeled `secure-agent` and is connected. |
| **Missing `API_TOKEN` value** | Credential ID `my-api-token` does not exist or is not accessible to the job. | Add the credential in **Jenkins > Credentials** (type: Secret Text) with ID `my-api-token`, and ensure the job has read permission. |
| **Workspace not cleaned** | `cleanWs()` step fails (e.g., locked files). | Check the build log for errors; ensure no background processes hold files, or add a `retry` wrapper around `cleanWs()`. |
| **Failure email not sent** | Mail plugin misconfigured or SMTP server unreachable. | Verify Jenkins **Manage Jenkins > Configure System > E‑mail Notification** settings and test with a simple `mail` step in a freestyle job. |

---

## Extending the Pipeline

When you are ready to add functional stages:

1. **Define stages** under the `stages` block.
2. **Use the environment variables** (`API_TOKEN`, `DISABLE_INSECURE_FEATURES`) directly in shell or Groovy steps.
3. **Maintain security** by keeping all secret handling inside the `environment` block or using `withCredentials`.

*Sample addition:*

```groovy
stages {
    stage('Checkout') {
        steps {
            checkout scm
        }
    }
    stage('Build') {
        steps {
            sh '''
                echo "Building with token $API_TOKEN"
                ./gradlew build
            '''
        }
    }
}
```

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.
- **Environment:** Secure token (`API_TOKEN`) and a hard‑coded safety flag.
- **Post actions:** Guaranteed workspace cleanup, audit logging, and failure notifications.
- **Current limitation:** No operational stages are defined; developers should add appropriate stages to meet project needs.

For any further customization or questions, consult the Jenkins Pipeline documentation or reach out to the DevOps team.