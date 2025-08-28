# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build process. Its primary objectives are:

- Execute builds in a restricted environment to reduce attack surface.  
- Capture and report build status for compliance and audit purposes.  
- Clean up the workspace securely after each run.  
- Notify the team on failures with limited, non‑sensitive details.

> **Note:** The `stages` section is currently empty. Add stages as needed for your project's build, test, and deployment steps.

---

## Agent Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| **Label** | `secure-agent` | Restricts the pipeline to run only on agents that have this label, ensuring a known, hardened execution environment. |

---

## Environment Variables

| Variable | Value / Source | Purpose |
|----------|----------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Securely injects an API token stored in Jenkins Credentials. Used by downstream steps that need to authenticate against external services. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | A flag that can be read by scripts to disable any optional insecure functionality. |

*All environment variables are automatically exported to each step of the pipeline.*

---

## Pipeline Stages

> **Current State:** No stages are defined (`"stages": []`).  
> To extend this pipeline, add stage blocks such as `Build`, `Test`, `Deploy`, etc., following the standard Declarative Pipeline syntax.

### Example Stage Skeleton

```groovy
stage('Build') {
    steps {
        // Insert build commands here
        sh 'make build'
    }
}
```

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

**Key Steps**

| Step | Command | Role |
|------|---------|------|
| `cleanWs()` | Jenkins built‑in step | Deletes all files in the workspace, ensuring no residual data remains. |
| `script { … }` | Groovy script block | Retrieves the final build status (`SUCCESS` if not set) and logs a concise audit message containing the build number and status. |

### `failure`

Executed **only** when the pipeline ends with a failure.

```groovy
// Notify on failure with restricted details
mail to: 'team@example.com',
     subject: "Build ${env.BUILD_NUMBER} Failed",
     body: "Check Jenkins for details: ${env.BUILD_URL}"
```

**Key Steps**

| Step | Command | Role |
|------|---------|------|
| `mail` | Jenkins email step | Sends a notification to the designated team address, providing the build number and a link to the Jenkins build page for further investigation. |

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Description |
|--------|-------------|
| **Manual** | Click **Build Now** on the pipeline job page in Jenkins. |
| **SCM Change** | Configure the job with a Git (or other SCM) webhook to trigger on push/PR events. |
| **Parameterized Build** | If you add parameters later, you can trigger via the **Build with Parameters** UI or API. |

### Monitoring Execution

1. **Console Output** – Click the build number in Jenkins to view real‑time logs.  
2. **Blue Ocean** – Use the Blue Ocean UI for a visual representation of stages (once stages are added).  
3. **Build Summary** – The `always` post block logs a concise audit line (`Build #X completed with status: Y`).  

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Build fails immediately with “No stages defined” | Empty `stages` array | Add at least one stage with steps that perform work. |
| Credential not found (`my-api-token`) | Missing or mis‑named credential in Jenkins | Verify the credential exists under **Credentials → System → Global credentials** and that the ID matches `my-api-token`. |
| Email not sent on failure | SMTP not configured or mail step mis‑configured | Check **Manage Jenkins → Configure System → E‑mail Notification** and ensure the `mail` step syntax matches your Jenkins version. |
| Workspace not cleaned | `cleanWs()` skipped due to early abort | Ensure the pipeline does not exit with `error` before reaching the `post` block, or add `catchError` around critical steps. |

---

## Extending the Pipeline

1. **Add Stages** – Insert stage blocks under the `stages` array to perform actual work (e.g., compile, test, package).  
2. **Use Environment Variables** – Reference `API_TOKEN` and `DISABLE_INSECURE_FEATURES` in shell or Groovy steps:  

   ```groovy
   sh """
   curl -H "Authorization: Bearer ${env.API_TOKEN}" https://api.example.com/endpoint
   """
   ```

3. **Secure Logging** – Replace the simple `echo` in the `always` block with calls to a centralized logging service if required.  

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.  
- **Environment:** Provides a protected API token and a flag to disable insecure features.  
- **Post‑Build:** Always cleans the workspace and logs status; on failure, sends a concise email alert.  
- **Next Steps:** Populate the `stages` section with project‑specific build, test, and deployment steps, and adjust post‑actions as needed.

---