---
layout: post
title: Cheapskate test reporting for Robot Framework
date: 2025-04-08 12:00:00
description: Test reporting for free with robotframework-dashboard
tags: [robotframework, qa, reporting, python, github, github-actions, dashboard]

---
I've been working for a client for past year and there hasn't been a decent test reporting services offered so some time ago I took a look at [robotframework-dashboard](https://github.com/timdegroot1996/robotframework-dashboard) as it seemed to tick most of my itches for getting statistics from my test runs.

My starting point; I run my test assets against 3 different environments: dev, uat & staging with firefox *and* chromium. In order to generate meaningful historical statistics i would need to:

* store output.xml for later processing
* store the resulting database after output.xml is processed
* have a backend to serve reports.

As GitHub offeres "github-pages", eg a service where one can host static html's, my needed list of functionality boils down to how and at what stage i would process output.xml and where to store the resulting database?

Since robotframework-dashboard supports sqlite as its backend, storing that database within repository ain't an bad option. At least not until the size of the sqlite file gets big enough but for now, storing it in git is good "poc" solution.

At this point as I already had my initial test workflow collecting all the necessary information my intial thought would be to make a separate workflow that gets triggered after actual testruns. One gotcha here thought. As explained earlier, 6 test runs in total nightly so I needed 6 different sqlite files and the newly created workflow had to know the parameters that where used to trigger the test run so that output.xml could be processed correctly

I solved this by generating a new file that i will store along the output.xml as build artifact:

```yaml
    - name: Install Dependencies & Generate env data
      run: |
        # irrelevant stuff removed
        mkdir -p reports
        echo  export BROWSER=${{ inputs.DEFAULT_BROWSER }} > reports/env.sh
        echo  export ENVIRONMENT=${{ inputs.TEST_ENV }} >> reports/env.sh
```

In this part, I'm storing the 2 arguments from workflow into separate shell script.

And then store the needed files as artifacts that could be later processed.

```yaml
    - name: COLLECT - Dashboard data
      uses: actions/upload-artifact@v4
      with:
        name: dashboarddata
        path: |
          reports/output.xml
          reports/env.sh
```

Both of these steps are part of the actual testrun workflow and are run in the same job as the tests. The next step is to create a new workflow that will be triggered after the testrun workflow has completed. This workflow will process the output.xml and generate and/or update the existing sqlite database.

```yaml
name: Generate Dashboard
on:
  workflow_run:
    workflows: ["Nightly / DEV / Chromium", "Nightly / DEV / Firefox", "Nightly / Staging / Chromium", "Nightly / Staging / Firefox", "Nightly / UAT / Chromium", "Nightly / UAT / Firefox"]
    types:
      - completed
```

This takes care of triggering. On `workflows:`, I listed all the jobs in my repository that will trigger this dashboard generation workflow.

In order to access all required features i needed during dashboard generation workflow, following permissions and settings need to be setup at the start of the workflow:

```yaml
jobs:
  process-artifact:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      pages: write
      actions: read
```
Environment is set to github-pages and url is provided from github. URL will depend on  repository access like is it on enterprise or public github but essentially its provided automatically.  First 3 permissions are important as those allows the workflow to write to the repository and update the github-pages branch. `actions: read` is needed to access the artifacts *if* repository is private.

Next step is to setup the workflow with actions dealing with pages:

```yaml
    steps:
    - name: Setup Pages
      uses: actions/configure-pages@v5
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Fetch all history for all branches and tags
        ref: "refs/heads/dashboard"
    - name: Set up Python 3.12
      uses: actions/setup-python@v5
      with:
        python-version: "3.12"
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip setuptools wheel
        python -m pip install  robotframework-dashboard
```

So, actions/configure-pages is the first step and should be self explanatory if you have dealt with github-pages before and then make sure there's working python and all the tools needed to process the output.xml and generate the sqlite database and pages. In this case, robotframework-dashboard is the only dependency needed.

But what about actions/checkout ? Im pulling in a branch from the same repository with fetch-depth set to 0. Why? I want to store my sqlite db files in that branch so that I don't need deal with persistency of data *or* handling of actual dev/production branches getting random commits from dashboard generation.

This `dashboard` branch was generated by creating "orphan" branch in the repository:

```bash
git checkout --orphan dashboard
```

This creates a new branch with no history and no files. On this branch i added a single html (`site/index.html`) that has combobox that will show 1 of the 6 generated  html files from robotframework-dashboard within iframe and 6 empty sqlite database files.

Next step in  workflow is to get the data:

```yaml
    - name: Download artifact from triggering workflow
      uses: actions/download-artifact@v4
      with:
        name: dashboarddata
        path: output/
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}
```

On this stage, passing `run-id` as that gives a correct "upstream" workflow id where we are downloading `dashboarddata` artifact from. The `path` is where the artifact will be downloaded to and `github-token` is needed to access the artifacts if the repository is private. And now we have both, output.xml and env.sh available for processing.

```yaml
    - name: Generate Dashboard
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
        done
```

And here is where the magic happens. First we source the generated env.sh so that we have BROWSER and ENVIRONMENT env variables set.  On initial call to robotdashboard, we are importing the data from output.xml into correct sqlite database and omit html generation. This is important because *everytime* this workflow is called, I need to generate all 6 html files so that deployment wont remove existing files. And thus, second call to robotdashboard is where we generate all those html files.

Next step, since we modified the sqlite database - we need to store that so that consecutive runs will always have previous data available.

```yaml
    - name: Commit and push changes
      run: |
        source ./output/env.sh
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"
        git add ./db/${ENVIRONMENT}_${BROWSER}.db
        git diff --cached --quiet || git commit -m "Processed artifact and updated files"
        git push origin HEAD:dashboard
```

And once again, we source the env.sh so that we know what database should have been modified. Rest of the steps are pretty standard git commands. We add the modified sqlite database and commit it. If there are no changes, we skip the commit step.

And finally, we need to deploy the generated html files to github-pages:

```yaml
    - name: Upload artifact
      uses: actions/upload-pages-artifact@v3
      with:
        # Upload entire repository
        path: site
    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

`actions/upload-pages-artifacts` gets `path:` set to site/, where all 6 newly created/updated dashboard files are along with the initial index.html that acts as selector for the 6 dashboards. And then we use `actions/deploy-pages` to deploy the site to github-pages.


---
