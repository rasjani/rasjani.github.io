---
layout: post
title: Cheapskate test reporting for Robot Framework
date: 2025-04-07 12:00:00
description: Test reporting for free with robotframework-dashboard
tags: [robotframework, qa, reporting, python, github, github-actions, dashboard]

---

### Cheapskate Test Reporting for Robot Framework

I've been working with a client over the past year, and one persistent issue has been the lack of a decent test reporting solution. After some digging, I stumbled upon robotframework-dashboard, which seemed to tick all the boxes I needed for extracting meaningful test run statistics.
#### My Setup

I run tests across three environments — dev, uat, and staging — using both Firefox and Chromium. To generate useful historical data, I needed to:

 * Store `output.xml` files for post-processing
 * Retain the resulting database after processing
 * Have a backend to serve reports

Since GitHub offers GitHub Pages for hosting static HTML content, the real question became: How and when should I process the `output.xml`, and where should I store the resulting database?
#### A Simple Storage Solution

Robotframework-dashboard supports SQLite, which made things simpler. I decided to store the SQLite database directly in the repo — not the most scalable option, but a solid proof-of-concept for now.

With my test workflow already collecting the right artifacts, I designed a second GitHub Actions workflow triggered after test runs. Here's the catch: six total test runs happen nightly (3 environments × 2 browsers), so I needed six separate SQLite files. The dashboard workflow needed to know which environment/browser combo each artifact belonged to.
#### Passing Parameters Between Workflows

I solved this by creating a simple shell script (env.sh) during the test run:
```yaml
- name: Install Dependencies & Generate env data
  run: |
    {% raw %} mkdir -p reports
    echo  export BROWSER=${{ inputs.DEFAULT_BROWSER }} > reports/env.sh
    echo  export ENVIRONMENT=${{ inputs.TEST_ENV }} >> reports/env.sh {% endraw %}
```
This stores the test parameters for the dashboard processor.

Then I uploaded both the test output and the environment metadata as artifacts:

```yaml
- name: COLLECT - Dashboard data
  uses: actions/upload-artifact@v4
  with:
    name: dashboarddata
    path: |
      reports/output.xml
      reports/env.sh
```

#### Creating the Dashboard Workflow

Now comes the second GitHub Actions workflow, which is triggered once the test workflows complete:

```yaml
{% raw %}name: Dashboard
name: Generate Dashboard
on:
  workflow_run:
    workflows: ["Nightly / DEV / Chromium", "Nightly / DEV / Firefox", "Nightly / Staging / Chromium", "Nightly / Staging / Firefox", "Nightly / UAT / Chromium", "Nightly / UAT / Firefox"]
    types:
      - completed
```

This ensures the dashboard updates after any of the nightly test runs.
#### Setup and Permissions

To generate and publish the dashboard, we need to configure permissions and environments:

```yaml
{% raw %}jobs:
  process-artifact:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      pages: write
      actions: read{% endraw %}
```
Next, configure the steps to check out the dashboard branch, set up Python, and install dependencies:

```yaml
steps:
  - name: Setup Pages
    uses: actions/configure-pages@v5
  - name: Checkout code
    uses: actions/checkout@v4
    with:
      fetch-depth: 0
      ref: "refs/heads/dashboard"
  - name: Set up Python 3.12
    uses: actions/setup-python@v5
    with:
      python-version: "3.12"
  - name: Install Dependencies
    run: |
      pip install --upgrade pip setuptools wheel
      pip install robotframework-dashboard
```
I use a dedicated `dashboard` branch (created as an orphan branch) to isolate dashboard files from main/test branches:

```bash
git checkout --orphan dashboard
```

This branch contains a basic index.html and six empty SQLite files.
#### Download Artifacts and Generate Dashboards

Now to process the test output:

```yaml
{% raw %}- name: Download artifact from triggering workflow
  uses: actions/download-artifact@v4
  with:
    name: dashboarddata
    path: output/
    run-id: ${{ github.event.workflow_run.id }}
    github-token: ${{ secrets.GITHUB_TOKEN }}{% endraw %}
```

Then we generate the database and HTML reports:

```yaml
{%raw %}- name: Generate Dashboard
  run: |
    source ./output/env.sh
    robotdashboard \
      -o output/output.xml \
      -d ./db/${ENVIRONMENT}_${BROWSER}.db \
      -g False

    for db in db/*.db; do
      x=$(basename "$db" .db)
      robotdashboard \
        -d ${db} \
        -t  "Nightly on ${x}" \
        -g True \
        -n site/${x}.html
    done{% endraw %}
```
First, we import test results into the correct .db. Then we regenerate all HTML files every time to avoid deleting existing reports during deployment.

#### Committing and Deploying

Next, commit and push the updated database:

```yaml
{% raw %}- name: Commit and push changes
  run: |
    source ./output/env.sh
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add ./db/${ENVIRONMENT}_${BROWSER}.db
    git diff --cached --quiet || git commit -m "Processed artifact and updated files"
    git push origin HEAD:dashboard{% endraw %}
```

And finally, deploy the updated dashboard:
```yaml
- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
  with:
    path: site
- name: Deploy to GitHub Pages
  id: deployment
  uses: actions/deploy-pages@v4
```
#### Final Thoughts

This setup gave me a clean, zero-cost solution for historical test reporting using Robot Framework and GitHub Actions. It’s not perfect, and storing .db files in the repo won’t scale forever, but it works great for now — especially for side projects or teams on a tight budget.

Let me know if you’re building something similar — always curious how others solve the same problems.

---
