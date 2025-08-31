# Setup Guide

This guide explains how to set up and test the **Jenkins Pipeline Automation Framework** inside GitHub Codespaces.

---

## 1. Fork Repository

- Fork [`jenkins-xaas`](https://github.com/rajan-s-verma/jenkins-xaas.git).
   - Keep repo name same (`jenkins-xaas`).
   - Fork only the **main** branch.

---

## 2. Create Client Pipelines Repository

1. Create a **private** repo named `jenkins-xaas-client-pipelines`.
   - Recommended: Select *Add README* to auto-create first commit and default `main` branch.

2. This repo will store generated Jenkinsfiles.
   - Folder structure: `<CLIENT_NAME>/Jenkinsfile`.

---

## 3. Open Codespaces

1. Open your fork (`jenkins-xaas`) in **GitHub Codespaces** (on branch `main`).
2. Wait for the devcontainer to start.
3. Open forwarded **port 8080** in the browser → this is your `JENKINS_URL`.
4. In the **Ports** tab of Codespaces, change the visibility of port **8080** from *Private* → *Public*.
   - This is required for deploying Client Job.

---

## 4. Jenkins First-Time Setup

1. Jenkins setup wizard will ask for the default password.
   Run in terminal:
   ```bash
   cat /var/jenkins_home/secrets/initialAdminPassword
   ```
2. Copy and paste into Jenkins setup page.
3. Install recommended plugins.
4. Create first admin user → this will be your login for all further Jenkins work.

---

## 5. Install Required Plugins
- Install all plugins listed in [plugins.md](plugins.md).
- After installing, restart Jenkins:
  - Open `<JENKINS_URL>/restart` → confirm.

---

## 6. Configure Markup Formatter
- Go to Manage Jenkins → Configure Global Security.
- Under Markup Formatter → select Safe HTML.
- Save.

---

## 7. Configure Credentials

### 7.1 GitHub SSH Key (github-ssh-creds)
- Generate SSH keypair inside Jenkins container:
  ```bash
  ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
  ```
- Default save path → `/var/jenkins_home/.ssh/id_rsa`.

- Add public key to GitHub
  - Copy contents of `/var/jenkins_home/.ssh/id_rsa.pub`
  - Go to Settings → SSH and GPG keys → New SSH Key.
  - Paste the key, add a descriptive title (e.g., Jenkins Server), and save.

- Add private key to Jenkins
  - In Jenkins: Manage Jenkins → Credentials → System → Global credentials → Add Credentials
  - Kind: SSH Username with private key
  - Username: git
  - Private Key: paste contents of `/var/jenkins_home/.ssh/id_rsa`
  - ID: github-ssh-creds

---

### 7.2 GitHub HTTPS Token (github-https-token)

- Generate Personal Access Token (classic) with:
    - repo
    - admin:repo_hook

- Add to Jenkins:
    - Kind: Username with password
    - Username: your GitHub username
    - Password: PAT
    - ID: github-https-token

---

### 7.3 Jenkins API Token (jenkins-api-token)

- In Jenkins → Your User → Security → API Token → Generate new token.

- Add to Jenkins:
    - Kind: Username with password
    - Username: your Jenkins username
    - Password: API token
    - ID: jenkins-api-token

---

## 8. GitHub Host Key Setup

After setting up the SSH key, run inside the devcontainer:

```bash
mkdir -p /var/jenkins_home/.ssh
ssh-keyscan github.com >> /var/jenkins_home/.ssh/known_hosts
chown 1000:1000 /var/jenkins_home/.ssh/known_hosts || true
```

---

## 9. Update Jenkinsfile in Repo

Edit `Jenkinsfile` in your fork:

- **environment{} block**
    ```groovy
    FORKED_BRANCH = 'main'     //Enter branch of your forked repo
    TARGET_BRANCH = 'main'     //Enker branch of repo created for storing Client Jenkinsfile in step 2
    ```
- **Stage: Checkout Generator Repository**
    ```groovy
    url: 'https://github.com/<YOUR_GITHUB_USERNAME>/jenkins-xaas'
    credsId: "${GITHUB_HTTPS_TOKEN}"  // GITHUB_HTTPS_TOKEN if using https url or GITHUB_SSH_TOKEN if using ssh url
    ```
- **Default REPO_URL parameter (Optional)**
    ```groovy
    defaultValue: 'https://github.com/<YOUR_GITHUB_USERNAME>/jenkins-xaas-client-pipelines'
    ```
---

## 10. Add Shared Library

Go to **Manage Jenkins → System → Global Pipeline Libraries**:

- **Name**: `jenkins-xaas-shared-lib`
- **Default version**: `main`
- **Retrieval**: Modern SCM → Git
- **Repo**: `https://github.com/<YOUR_GITHUB_USERNAME>/jenkins-xaas`
- **Credentials**: `github-https-token`.

---

## 11. Create Client Onboarding Master Job

1. Go to **New Item → Pipeline**.
2. Enter **Name**: `Client-Onboarding-Master`.
3. Configure pipeline to use Jenkinsfile at:
```text
Jenkinsfile
```
4. In Job Configuration, add input parameters:

   - `CLIENT_NAME` → new client name
   - `REPO_URL` → `https://github.com/<YOUR_GITHUB_USERNAME>/jenkins-xaas-client-pipelines`
   - ⚠️ **Note:** If parameters are not configured here, the first build will fail.

5. Trigger the first build using **Build with Parameters**, providing values for `CLIENT_NAME` and `REPO_URL`.

---

## 12. Verify

- Generated Jenkinsfiles appear in:
  ```text
  jenkins-xaas-client-pipelines/<CLIENT_NAME>/Jenkinsfile
  ```
---

## 13. Quick Smoke Test

Run the included smoke test to verify everything works:

```bash
cd pipeline-generator
./tests/smoke_test.sh
```
- If successful, the script will finish without errors and confirm Jenkins can generate pipelines.
---

## 14. Troubleshooting

- If Jenkins doesn’t start → Rebuild the Codespace.
- If GitHub cloning fails → Ensure `github-ssh-creds` or `github-https-token` is set correctly.
- If jobs don’t deploy → Check `jenkins-api-token`, Jenkins URL in job configs and visibility of port 8080 should be `Public`.

---

## 15. Reference Screenshots

To illustrate the setup and workflow, here are the screenshots:

1. |[Jenkins home/dashboard in Codespaces](./images/01-devcontainer-jenkins-home.png)
2. |[Installed plugins](./images/02-jenkins-plugins.png)
3. |[Configured credentials](./images/03-jenkins-credentials.png)
4. |[Client Onboarding Master Job](./images/04-created-client-job.png)
5. |[Auto created client Job](./images/05-auto-created-client-job.png)
6. |[Auto generated client Jenkinsfile in GitHub](./images/06-auto-deployed-client-pipeline.png)
---

✅ With this setup, you’ll have a working Jenkins Pipeline Automation Framework running inside Codespaces.
