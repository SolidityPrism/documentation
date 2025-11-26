# Solidity Prism Documentation

Welcome to the official documentation for Solidity Prism. Here you will find everything you need to install, configure, and use our AI-powered audit tool.

***

### Table of Contents
1. [How It Works](#how-it-works)
2. [GitHub App Installation](#github-app-installation)
3. [Workflow Installation](#workflow-installation)
4. [Usage](#usage)
5. [Uninstalling](#uninstalling)
6. [Support](#support)

***

## How It Works

Solidity Prism is a GitHub Action that runs as an automated code reviewer on your Pull Requests.

It does **not** run on every single push. Instead, you trigger it on demand by posting a comment on a Pull Request. This keeps your workflow clean and ensures audits are only run when you are ready.

Once triggered, the action sends your code to our secure backend for analysis and posts a summary report back as a comment.

***

## GitHub App Installation

> **Important:**  
> The Solidity Prism Auditor GitHub App is installed **just once per owner or organization** – not per repository.  
> 
> After installation, you can select which repositories in your organization (or under your GitHub account) will grant access to the app.  
> You can always change the selection and add/remove specific repositories later from the same installation page.
>
> **For organizations:** Only admins can install or configure app access.

### How to Install the GitHub App
<a href="https://github.com/organizations/SolidityPrism/settings/apps/solidity-prism-auditor/installations">
  <img src="https://img.shields.io/badge/Install%20App-blue?style=for-the-badge" alt="Install Solidity Prism Auditor">
</a>

<button onclick="window.open('https://github.com/organizations/SolidityPrism/settings/apps/solidity-prism-auditor/installations', '_blank')">
  Install Solidity Prism Auditor on your organization
</button>


1. Open this link in a new tab:  
   <a href="https://github.com/organizations/SolidityPrism/settings/apps/solidity-prism-auditor/installations" target="_blank">Install Solidity Prism Auditor on your organization</a>
2. Select the repositories you want to grant access to:
   - **All repositories** – The app will have access to all current and future repositories.
   - **Only select repositories** – Choose specific repositories manually.
3. Click **Save** to confirm the installation.
4. The app is now installed and ready to post audit results to Pull Requests.

**Example screenshot:**

![App install and repo selection](image Uninstall the GitHub App

To modify which repositories have access, or to uninstall the app:

1. Open this link:  
   <a href="https://github.com/organizations/SolidityPrism/settings/apps/solidity-prism-auditor/installations" target="_blank">Manage Solidity Prism Auditor Installation</a>
2. Click **Configure** next to the app.
3. Either:
   - Update the repository selection (add/remove specific repositories)
   - Click **Uninstall** to remove the app entirely from your organization

***

## Workflow Installation

After installing the GitHub App, you must add the GitHub Action workflow to each repository where you want to use Solidity Prism audits.

Getting started takes less than 5 minutes.

### Step 1: Create the Workflow File

In your project's repository, create a new file at this path: `.github/workflows/solidity_prism.yml`.

### Step 2: Paste the Workflow Configuration

Copy and paste the following code into the `solidity_prism.yml` file you just created:

```yaml
# .github/workflows/solidity_prism.yml

name: 'Solidity Prism Audit'

on:
  issue_comment:
    types: [created]

jobs:
  run-audit:
    # This job only runs when a PR comment starts with "/audit"
    if: github.event.issue.pull_request && startsWith(github.event.comment.body, '/audit')
    permissions:
      pull-requests: write
    runs-on: ubuntu-latest
    steps:
      # Step 1: Parse the command from the comment (e.g., /audit gas deep)
      - name: Parse Command
        id: parse_command
        run: |
          COMMENT="${{ github.event.comment.body }}"
          AUDIT_TYPE=$(echo "$COMMENT" | awk '{print $2}')
          if [ -z "$AUDIT_TYPE" ]; then AUDIT_TYPE="full"; fi
          ANALYSIS_MODE=$(echo "$COMMENT" | awk '{print $3}')
          if [ -z "$ANALYSIS_MODE" ]; then ANALYSIS_MODE="standard"; fi
          echo "audit_type=$AUDIT_TYPE" >> $GITHUB_OUTPUT
          echo "analysis_mode=$ANALYSIS_MODE" >> $GITHUB_OUTPUT

      # Step 2: Get the exact code from the Pull Request
      - name: Get PR's Head SHA
        id: get_sha
        run: |
          PR_DATA=$(curl -s -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" ${{ github.event.issue.pull_request.url }})
          HEAD_SHA=$(echo $PR_DATA | jq -r '.head.sha')
          echo "sha=$HEAD_SHA" >> $GITHUB_OUTPUT
      
      # Step 3: Trigger the Solidity Prism backend API
      - name: Trigger Audit Backend
        run: |
          curl -X POST "https://audit.solidityprism.dev/api/v1/github-action-trigger" \
          -H "Content-Type: application/json" \
          -H "X-Action-Secret: ${{ secrets.PRISM_ACTION_SECRET }}" \
          -d '{
            "repository_id": "${{ github.event.repository.id }}",
            "repository_name": "${{ github.event.repository.full_name }}",
            "owner_id": "${{ github.event.repository.owner.id }}",
            "repository_url": "${{ github.event.repository.git_url }}",
            "pr_number": "${{ github.event.issue.number }}",
            "commit_sha": "${{ steps.get_sha.outputs.sha }}",
            "comments_url": "${{ github.event.issue.comments_url }}",
            "github_token": "${{ github.token }}",
            "audit_type": "${{ steps.parse_command.outputs.audit_type }}",
            "analysis_mode": "${{ steps.parse_command.outputs.analysis_mode }}"
          }'
```

### Step 3: Activate Repository & Add Secret

For the action to securely communicate with our backend, you need to activate your repository and add a secret to GitHub.

1. Log in to your account on <a href="https://solidityprism.dev/dashboard" target="_blank">**solidityprism.dev/dashboard**</a>
2. Find your repository in the list.

> **⚠️ Don't see your Organization's repositories?**
>
> If you are part of an organization, GitHub often restricts third-party access by default. To show your repositories:
> 1. Go to your **Personal GitHub Settings** (click your avatar top-right > **Settings**).
> 2. On the left sidebar, click **Applications** (at the bottom) > **Authorized OAuth Apps**.
> 3. Click on **Solidity Prism**.
> 4. Scroll down to the **Organization access** section.
> 5. Click **Grant** (or **Request**) next to your organization's name.
>
> Refresh your dashboard, and the repositories will appear.

3. Click the **Activate & Buy Credits** button (if not already active).
4. Once the repository status is **Active**, click the purple **Key** button next to it.

![Activated Repository and Key Button](images. Copy the displayed `PRISM_ACTION_SECRET`.
6. In your GitHub repository, go to **Settings > Secrets and variables > Actions**.
7. Click the **New repository secret** button.
8. **Name:** `PRISM_ACTION_SECRET`
9. **Value:** Paste the key you copied.

That's it! Solidity Prism is now installed and ready to audit.

***

## Usage

To trigger an audit, go to any open Pull Request and post a comment with one of the following commands. You can also specify the analysis depth: `fast`, `standard` (default), or `deep`.

| Command                 | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `/audit`                | Runs a **full** audit (security + gas) in **standard** mode. |
| `/audit security`       | Runs a **security only** audit in **standard** mode.         |
| `/audit gas`            | Runs a **gas only** audit in **standard** mode.              |
| `/audit security deep`  | Runs a **security only** audit in **deep** mode.             |
| `/audit gas fast`       | Runs a **gas only** audit in **fast** mode.                  |

The analysis depth always comes last in the command.

***

## Uninstalling

### Removing the Workflow (Per Repository)

If you want to stop using Solidity Prism audits in a specific repository:

1. In your repository, navigate to `.github/workflows/solidity_prism.yml`
2. Delete this file and commit the change
3. This will immediately stop the action from running on future Pull Requests

### Remove the Secret (Optional but Recommended)

1. Go to your repository **Settings → Secrets and variables → Actions**
2. Find `PRISM_ACTION_SECRET` in the list
3. Click the **Remove** button next to it
4. Confirm deletion

### Deactivate Repository in Dashboard (Optional)

1. Log in to your **Solidity Prism Dashboard** on <a href="https://solidityprism.dev/dashboard" target="_blank">solidityprism.dev/dashboard</a>
2. Find the repository you want to deactivate
3. Click **Deactivate** or **Remove**

**Note:** Deleting the workflow file is sufficient to stop audits. The remaining steps ensure complete cleanup.

***

### Uninstalling the GitHub App (Organization-Wide)

To fully remove the Solidity Prism Auditor GitHub App from your organization or personal account:

1. Open this link:  
   <a href="https://github.com/organizations/SolidityPrism/settings/apps/solidity-prism-auditor/installations" target="_blank">Manage Solidity Prism Auditor Installation</a>
2. Click **Configure** next to the app.
3. Choose to:
   - Remove access from specific repositories
   - Click **Uninstall** to remove the app entirely
4. Confirm the action.

After uninstalling, the app will no longer have access or be able to comment on Pull Requests in any repository.

***

## Support

If you have any questions, encounter issues, or have a feature request, please <a href="https://github.com/SolidityPrism/documentation/issues" target="_blank">open an issue</a> in this repository.
