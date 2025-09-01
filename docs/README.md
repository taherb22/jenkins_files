# Jenkins Pipeline Documentation

## Overview

**Pipeline Name:** *Unnamed (provided JSON)*  
**Agent:** `label 'secure-agent'` – the pipeline runs on a Jenkins node that matches the `secure-agent` label.

**Purpose:**  
This pipeline validates that a branch name is supplied before any further build or deployment steps are executed. It is intended to be used as a guard stage in larger CI/CD workflows, ensuring that downstream stages receive a valid `BRANCH_NAME` parameter.

---

## Table of Contents
1. [Pipeline Summary](#pipeline-summary)  
2. [Stages](#stages)  
   - [Initialize](#initialize)  
3. [Environment Variables](#environment-variables)  
4. [Running the Pipeline](#running-the-pipeline)  
5. [Monitoring & Logs](#monitoring--logs)  
6. [Troubleshooting](#troubleshooting)  
7. [Missing or Optional Sections](#missing-or-optional-sections)  

---

## Pipeline Summary
The pipeline consists of a single **Initialize** stage that performs a sanity check on the `BRANCH_NAME` parameter. If the parameter is missing or empty, the pipeline aborts with a clear error message. The environment is pre‑populated with an API token (secured via Jenkins credentials) and a flag to disable insecure features.

---

## Stages

### Initialize
| Attribute | Value |
|-----------|-------|
| **When** | No conditional `when` clause – the stage always runs. |
| **Purpose** | Ensure a valid `BRANCH_NAME` is provided before any further processing. |
| **Key Activities** | Groovy script that validates the parameter and fails fast if the check does not pass. |

#### Detailed Steps
```groovy
if (params.BRANCH_NAME == null || params.BRANCH_NAME.trim().isEmpty()) {
    error 'Branch name is required and cannot be empty'
}
```
* **What it does**  
  * Checks the `BRANCH_NAME` parameter supplied to the build.  
  * Trims whitespace and verifies the value is not `null` or an empty string.  
  * Calls `error` to abort the pipeline with a descriptive message when the validation fails.

* **Result**  
  * **Success:** Pipeline proceeds to subsequent stages (if any).  
  * **Failure:** Build is marked **FAILED** and stops immediately, preventing downstream actions.

---

## Environment Variables

| Variable | Definition | Purpose |
|----------|------------|---------|
| `API_TOKEN` | `credentials('my-api-token')` | Retrieves a secret API token from Jenkins Credentials Store. The token is injected as a masked environment variable for use by downstream steps (e.g., API calls). |
| `DISABLE_INSECURE_FEATURES` | `'true'` | A static flag indicating that insecure features should be disabled. Downstream scripts can read this variable to enforce stricter security behavior. |

*All environment variables are defined at the top level of the pipeline and are available to every stage.*

---

## Running the Pipeline

1. **Triggering Manually**  
   - Navigate to the Jenkins job page.  
   - Click **Build with Parameters**.  
   - Provide a value for **BRANCH_NAME** (required).  
   - Click **Build**.

2. **Triggering via SCM/Webhook**  
   - If the job is configured with a multibranch pipeline or Git webhook, Jenkins will automatically pass the branch name as `BRANCH_NAME`. Ensure the webhook payload includes the branch reference.

3. **Using the Jenkinsfile**  
   - Place the provided pipeline definition in a `Jenkinsfile` at the root of your repository.  
   - Commit and push; the pipeline will be picked up according to your job’s SCM configuration.

---

## Monitoring & Logs

- **Console Output:**  
  Access the build’s **Console Output** from the Jenkins UI to view the validation step and any error messages.

- **Blue Ocean (optional):**  
  If Blue Ocean is installed, you can view a visual representation of the stage flow and quickly identify where the pipeline stopped.

- **Environment Variable Visibility:**  
  Sensitive values (e.g., `API_TOKEN`) are masked in logs. Non‑sensitive variables like `DISABLE_INSECURE_FEATURES` will appear as plain text.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Build fails with *“Branch name is required and cannot be empty”* | `BRANCH_NAME` parameter not supplied or empty. | Provide a non‑empty branch name when triggering the build. |
| Build aborts before reaching the Initialize stage | Agent with label `secure-agent` unavailable. | Verify that at least one Jenkins node is labeled `secure-agent` and is online. |
| `API_TOKEN` appears as `<masked>` in logs but downstream steps report authentication failures. | Credential ID `my-api-token` is missing or has incorrect permissions. | Check **Jenkins → Credentials** for the correct ID and ensure the token is valid. |
| Environment variable `DISABLE_INSECURE_FEATURES` not recognized in downstream scripts. | Variable not exported or referenced incorrectly. | Use `${env.DISABLE_INSECURE_FEATURES}` (Groovy) or `$DISABLE_INSECURE_FEATURES` (shell) to read the value. |

---

## Missing or Optional Sections

- **Post Actions:** The `post` block is empty. Consider adding notifications (e.g., Slack, email) or cleanup steps as needed.
- **Additional Stages:** Only the `Initialize` stage is defined. In a full CI/CD pipeline, you would typically follow this with build, test, and deploy stages that consume the validated `BRANCH_NAME`.
- **Parameters Declaration:** The JSON does not include a `parameters` block. Adding a `string` parameter for `BRANCH_NAME` would improve UI ergonomics and documentation. Example:

```groovy
parameters {
    string(name: 'BRANCH_NAME', description: 'Git branch to build', defaultValue: '')
}
```

---

*End of documentation.*