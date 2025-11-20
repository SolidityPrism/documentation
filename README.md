# Solidity Prism Documentation

Welcome to the official documentation for Solidity Prism. Here you will find everything you need to install, configure, and use our AI-powered audit tool.

---

### Table of Contents
1. [How It Works](#how-it-works)
2. [Installation](#installation)
3. [Usage](#usage)
4. [Uninstalling](#uninstalling)
5. [Support](#support)

---

## How It Works

Solidity Prism is a GitHub Action that runs as an automated code reviewer on your Pull Requests.

It does **not** run on every single push. Instead, you trigger it on demand by posting a comment on a Pull Request. This keeps your workflow clean and ensures audits are only run when you are ready.

Once triggered, the action sends your code to our secure backend for analysis and posts a summary report back as a comment.

## Installation

Getting started with Solidity Prism takes less than 5 minutes.

### Step 1: Create the Workflow File

In your project's repository, create a new file in the following path: `.github/workflows/solidity_prism.yml`.

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

### Step 3: Add the Secret

For the action to securely communicate with our backend, you need to add a secret to your repository.

1.  Log in to your account on **solidityprism.dev**.
2.  Go to your **Dashboard** or **Account Settings** page.
3.  You will find your unique `PRISM_ACTION_SECRET` key there. Copy it.
4.  In your GitHub repository, go to **Settings > Secrets and variables > Actions**.
5.  Click the **New repository secret** button.
6.  **Name:** `PRISM_ACTION_SECRET`
7.  **Value:** Paste the key you copied from your dashboard.

That's it! Solidity Prism is now installed.

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

## Uninstalling

If you want to stop using Solidity Prism or revoke its access to your repository, follow these steps:

### Step 1: Delete the Workflow File

1. In your repository, navigate to `.github/workflows/solidity_prism.yml`
2. Delete this file and commit the change
3. This will immediately stop the action from running on future Pull Requests

### Step 2: Remove the Secret (Optional but Recommended)

1. Go to your repository **Settings → Secrets and variables → Actions**
2. Find `PRISM_ACTION_SECRET` in the list
3. Click the **Remove** button next to it
4. Confirm deletion

### Step 3: Deactivate Repository in Dashboard (Optional)

1. Log in to your **Solidity Prism Dashboard** on [solidityprism.dev](https://solidityprism.dev)
2. Go to your **Repositories** page
3. Find the repository you want to deactivate
4. Click **Deactivate** or **Remove**

**Note:** Deleting the workflow file is sufficient to stop audits. The remaining steps ensure complete cleanup.

## Support

If you have any questions, encounter issues, or have a feature request, please [open an issue](https://github.com/SolidityPrism/documentation/issues) in this repository.
