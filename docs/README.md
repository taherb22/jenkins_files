# Jenkins Pipeline Documentation

## Overview

This pipeline is designed to run on a **controlled, secure agent** (`secure-agent`) and provides a minimal, auditable build process. Its primary objectives are:

- Execute builds in a restricted environment to reduce attack surface.  
- Capture build status and audit information for compliance.  
- Perform secure workspace cleanup after every run.  
- Notify the team on failure while limiting exposed details.

> **Note:** The `stages` section is empty in the supplied definition. If additional build, test, or deployment stages are required, they should be added under the `stages` array.

---

## Agent Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| **Label** | `secure-agent` | Restricts the pipeline to run only on agents that carry this label, ensuring a known, hardened execution environment. |

---

## Environment Variables

| Variable | Source | Default / Value | Purpose |
|----------|--------|----------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | (retrieved from Jenkins Credentials Store) | Securely provides an API token for any external service calls required by the pipeline. |
| `DISABLE_INSECURE_FEATURES` | `'true'` | `true` | Global flag used by downstream scripts to disable any functionality deemed insecure. |

*All environment variables are injected automatically at the start of the pipeline and are available to every step.*

---

## Pipeline Stages

> **Current State:** No stages are defined (`"stages": []`).  
> To extend the pipeline, add stage blocks such as `stage('Build') { steps { … } }`, `stage('Test') { … }`, etc.

---

## Post‑Build Actions

Post actions run after the main pipeline execution, regardless of success or failure.

### `always`

Executed after every run.

1. **Secure Workspace Cleanup**  
   ```groovy
   cleanWs()
   ```
   *Removes all files from the workspace, ensuring no residual data remains.*

2. **Audit Log Emission**  
   ```groovy
   script {
       def buildStatus = currentBuild.result ?: 'SUCCESS'
       echo "Build ${env.BUILD_NUMBER} completed with status: ${buildStatus}"
   }
   ```
   *Logs the build number and final status (`SUCCESS`, `FAILURE`, etc.) to the Jenkins console, which can be forwarded to external logging services.*

### `failure`

Executed **only** when the pipeline ends with a failure.

1. **Restricted Failure Notification**  
   ```groovy
   mail to: 'team@example.com',
        subject: "Build ${env.BUILD_NUMBER} Failed",
        body: "Check Jenkins for details: ${env.BUILD_URL}"
   ```
   *Sends an email to the designated team address with a minimal payload, directing recipients to the Jenkins UI for full details.*

---

## Usage Instructions for Developers

### Triggering the Pipeline

| Method | Description |
|--------|-------------|
| **Manual** | Click **Build Now** on the pipeline job page in Jenkins. |
| **SCM Change** | Configure a webhook or poll SCM to trigger on commits/PRs. |
| **Parameterized Build** | (If added later) Use the **Build with Parameters** UI to pass custom values. |

### Monitoring Execution

1. **Console Output** – Click the build number → **Console Output** to view real‑time logs, including the audit echo from the `always` block.  
2. **Blue Ocean** – For a visual stage view (once stages are added).  
3. **Build History** – Use the Jenkins dashboard to see recent runs and their statuses.

### Troubleshooting Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| **Pipeline fails before any stage** | Agent label not available or offline. | Verify that at least one agent is labeled `secure-agent` and is online. |
| **Missing `API_TOKEN`** | Credential ID typo or missing credential. | Ensure a credential with ID `my-api-token` exists in **Jenkins → Credentials** and is accessible to the pipeline job. |
| **Email not sent on failure** | SMTP configuration error or mail step disabled. | Check Jenkins global mail settings and confirm the `mail` step is allowed in the sandbox (or run in an approved script). |
| **Workspace not cleaned** | `cleanWs()` step skipped due to early abort. | Ensure the pipeline reaches the `post` section; consider adding `catchError` blocks to handle early failures gracefully. |

---

## Extending the Pipeline

When additional functionality is needed:

1. **Add Stages** – Insert stage blocks under the `stages` array, e.g.:

   ```groovy
   stages {
       stage('Build') {
           steps {
               sh 'make build'
           }
       }
       stage('Test') {
           steps {
               sh 'make test'
           }
       }
   }
   ```

2. **Inject Additional Environment Variables** – Extend the `environment` block with new entries, using either static values or Jenkins credentials.

3. **Custom Post Actions** – Add more conditions (`success`, `unstable`, `aborted`) under `post` as required.

---

## Summary

- **Agent:** Runs exclusively on `secure-agent`.  
- **Environment:** Secure token (`API_TOKEN`) and a flag to disable insecure features.  
- **Post‑Build:** Guarantees workspace cleanup, logs build status, and notifies the team on failure.  
- **Current Gaps:** No defined stages; developers should add build, test, and deployment steps as needed.

For any further customization or questions, consult the Jenkinsfile reference or reach out to the DevOps team.