# Solidity Prism Documentation

Welcome to the official documentation for Solidity Prism. Here you will find everything you need to install, configure, and use our AI-powered audit tool.

---

### Table of Contents
1. [How It Works](#how-it-works)
2. [Installation](#installation)
3. [Usage](#usage)
4. [Support](#support)

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
            "repository_url": "${{ github.repositoryUrl }}",
            "pr_number": "${{ github.event.issue.number }}",
            "commit_sha": "${{ steps.get_sha.outputs.sha }}",
            "comments_url": "${{ github.event.issue.comments_url }}",
            "github_token": "${{ secrets.GITHUB_TOKEN }}",
            "audit_type": "${{ steps.parse_command.outputs.audit_type }}",
            "analysis_mode": "${{ steps.parse_command.outputs.analysis_mode }}"
          }'

```

### Step 3: Add the Secret

For the action to securely communicate with our backend, you need to add a secret to your repository.

1.  In your repository, go to **Settings > Secrets and variables > Actions**.
2.  Click the **New repository secret** button.
3.  **Name:** `PRISM_ACTION_SECRET`
4.  **Value:** Contact us at `votre-email@exemple.com` to get your private key. 
    *(Note: This is a placeholder for how you will onboard users in the future).*

That's it! Solidity Prism is now installed.

## Usage

To trigger an audit, simply go to any open Pull Request and post a comment with one of the following commands:

| Command                 | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `/audit`                | Runs a **full** audit (security + gas) in **standard** mode. |
| `/audit security`       | Runs a **security only** audit in **standard** mode.         |
| `/audit gas`            | Runs a **gas only** audit in **standard** mode.              |
| `/audit full deep`      | Runs a **full** audit in **deep** analysis mode.             |
| `/audit security fast`  | Runs a **security only** audit in **fast** analysis mode.    |


## Support

If you have any questions, encounter issues, or have a feature request, please [open an issue](https://github.com/SolidityPrism/documentation/issues) in this repository.
