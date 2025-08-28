# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build flow. Its primary objectives are:

- Execute builds on a restricted agent to enforce security boundaries.  
- Supply required credentials and configuration via environment variables.  
- Perform deterministic post‑build actions: workspace cleanup, audit logging, and failure notifications.

> **Note:** The `stages` section is empty in the supplied definition. If additional build stages are required, they should be added under the `stages` block following the same pattern used for the post actions.

---

## Agent Configuration

```groovy
label 'secure-agent'
```

- **Purpose:** Guarantees that the pipeline runs only on nodes tagged with `secure-agent`.  
- **Effect:** Limits exposure to only vetted machines, reducing attack surface.

---

## Environment Variables

| Variable | Definition | Purpose |
|----------|------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Retrieves a secret API token from Jenkins Credentials Store. The token is injected into the build environment for any API calls that require authentication. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | A flag used by downstream scripts to disable any functionality that is considered insecure. Should be respected by all scripts executed in the pipeline. |

*All environment variables are automatically exported for every step in the pipeline.*

---

## Stages

> **Current State:** No stages are defined (`"stages": []`).  
> To extend the pipeline, add stage blocks such as:

```groovy
stage('Build') {
    steps {
        // build commands
    }
}
```

Each stage should include a clear purpose, required steps, and any artifact handling.

---

## Post‑Build Actions

Post actions run after the main pipeline execution, regardless of success or failure.

### `always`

Executed after every run.

```groovy
// Clean up workspace securely
cleanWs()

// Send audit log to a secure logging service
script {
    def buildStatus = currentBuild.result ?: 'SUCCESS'
    echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
}
```

- **`cleanWs()`** – Removes all files from the workspace, ensuring no residual data remains on the agent.  
- **Audit Logging Script** – Captures the final build status (`SUCCESS` by default) and logs a concise message containing the build number and status. This output can be forwarded to an external logging service via Jenkins log aggregation.

### `failure`

Executed **only** when the build fails.

```groovy
// Notify on failure with restricted details
mail to: 'team@example.com',
     subject: "Build ${env.BUILD_NUMBER} Failed",
     body: "Check Jenkins for details: ${env.BUILD_URL}"
```

- **Email Notification** – Sends a minimal failure alert to the designated team address. The email includes the build number and a direct link to the Jenkins build page for further investigation.

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Description |
|--------|-------------|
| **Manual** | Click **Build Now** on the pipeline job page in Jenkins. |
| **SCM Hook** | Configure a webhook (e.g., GitHub, GitLab) to trigger the job on push/PR events. |
| **Parameterized Build** | If parameters are added later, use the **Build with Parameters** UI or the Jenkins REST API. |

### Monitoring Execution

1. **Console Output** – Click the build number to view real‑time logs.  
2. **Blue Ocean** – Use the Blue Ocean UI for a visual representation of stages (once stages are defined).  
3. **Build Summary** – The `always` post block logs a concise status line (`Build #X completed with status: Y`).  

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Workspace not cleaned** | `cleanWs()` step skipped due to early abort. | Ensure the pipeline reaches the `post` section; avoid `System.exit` or `return` statements before the end. |
| **Missing API token** | Credential ID `my-api-token` not defined or inaccessible. | Verify the credential exists in **Jenkins > Credentials** and that the pipeline’s job has read access. |
| **Failure email not sent** | SMTP configuration error or mail step disabled. | Check **Manage Jenkins > Configure System > E-mail Notification** and confirm the `mail` step is allowed by the security sandbox. |
| **`DISABLE_INSECURE_FEATURES` ignored** | Downstream scripts do not read the variable. | Update scripts to reference `env.DISABLE_INSECURE_FEATURES` and enforce the flag. |

---

## Extending the Pipeline

When adding new stages:

1. **Define a stage block** with a descriptive name.  
2. **Add `steps`** that perform the required actions (e.g., `sh`, `bat`, `script`).  
3. **Use environment variables** (`API_TOKEN`, `DISABLE_INSECURE_FEATURES`) as needed.  
4. **Optionally add `post` actions** inside the stage for stage‑specific cleanup or reporting.

Example skeleton:

```groovy
stage('Test') {
    steps {
        sh '''
            echo "Running tests..."
            ./run-tests.sh --token $API_TOKEN
        '''
    }
    post {
        success {
            echo 'Tests passed.'
        }
        failure {
            mail to: 'qa@example.com',
                 subject: "Test stage failed in build ${env.BUILD_NUMBER}",
                 body: "See ${env.BUILD_URL} for details."
        }
    }
}
```

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.  
- **Environment:** Supplies a credential (`API_TOKEN`) and a security flag (`DISABLE_INSECURE_FEATURES`).  
- **Post‑Build:** Guarantees workspace cleanup, logs build status, and notifies the team on failures.  
- **Current Gaps:** No functional stages are defined; developers should add appropriate stages to meet project needs.

For any further customization or questions, consult the Jenkins administrator or the DevOps team.