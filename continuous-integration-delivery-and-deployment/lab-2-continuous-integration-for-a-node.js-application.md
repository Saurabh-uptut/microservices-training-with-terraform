# Lab 2: Continuous Integration for a Node.js Application

Build, Test, and Publish Artifacts Using GitHub Actions

This lab walks you through creating a complete CI workflow for a Node.js application.\
The workflow will install dependencies, run tests, generate reports, and publish build artifacts (including a deployment-ready ZIP file).

***

### Pre-Requisite

* A GitHub account
* Basic familiarity with Git and GitHub repositories

***

### Learning Objectives

1. Understand what Continuous Integration (CI) is and why teams use it.
2. Create a GitHub Actions workflow for a Node.js application.
3. Learn how to build and test a Node.js app automatically.
4. Publish test results and application artifacts using GitHub Actions.
5. Gain confidence in reading and modifying CI YAML workflows.

***

### Step 1: Fork and Clone the Application

1. Fork the sample application repository provided by your instructor.
2. Clone the repository to your local machine.

```bash
git clone <repository-url>
cd <repository-folder>
```

***

### Step 2: Create Workflow Directory

Inside the cloned project, create the required GitHub Actions folder:

```
.github/workflows
```

Inside this folder, create a file named:

```
ci.yml
```

***

### Step 3: Add the CI Pipeline Code

Copy the following YAML code into ci.yml.

```
name: Continuous Integration Pipeline

on:
  workflow_dispatch:

permissions:
  contents: read
  actions: read
  checks: write
  pull-requests: write

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: 'package-lock.json'

      - name: Install dependencies
        run: npm ci

      - name: Build the application (optional)
        run: |
          npm run build 2>/dev/null || echo "No build script found, skipping build step"

      - name: Run unit tests with coverage and JUnit reports
        env:
          JEST_JUNIT_OUTPUT: coverage/junit.xml
        run: npm run test:ci

      - name: Publish test results to Checks
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Jest Test Results
          path: coverage/junit.xml
          reporter: jest-junit
          fail-on-error: true

      - name: Upload test results to GitHub
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-node-${{ matrix.node-version }}
          path: coverage/
          retention-days: 30

      - name: Create deployment package
        run: |
          set -euo pipefail
          STAGING="$GITHUB_WORKSPACE/deployment-package"
          rm -rf "$STAGING"
          mkdir -p "$STAGING"

          [ -d public ] && cp -r public "$STAGING/" || echo "No public directory found"
          [ -d node_modules ] && cp -r node_modules "$STAGING/" || echo "No node_modules directory found"
          [ -f server.js ] && cp server.js "$STAGING/" || echo "No server.js found"
          cp package.json "$STAGING/"
          cp package-lock.json "$STAGING/"
          [ -f README.md ] && cp README.md "$STAGING/" || true
          [ -f Dockerfile ] && cp Dockerfile "$STAGING/" || echo "No Dockerfile found"

          (cd "$STAGING" && npm pkg delete devDependencies || true)

          TS="$(date +%Y%m%d-%H%M%S)"
          SHORT_SHA="${GITHUB_SHA::7}"
          ZIP_NAME="deployment-package-${SHORT_SHA}-${TS}.zip"
          (cd "$GITHUB_WORKSPACE" && zip -r "$ZIP_NAME" "deployment-package")
          echo "ZIP_NAME=$ZIP_NAME" >> "$GITHUB_ENV"

      - name: Upload deployment package artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-package-node-${{ matrix.node-version }}-${{ github.run_number }}
          path: ${{ env.ZIP_NAME }}
          retention-days: 90
```

***

### Step 4: Understanding the Pipeline

#### What this CI pipeline does

* Builds and tests your Node.js application
* Publishes JUnit test results to GitHub Checks
* Saves two types of artifacts:
  1. Test results and coverage reports
  2. A deployment-ready ZIP package of your app
* Runs the workflow across Node.js versions 18 and 20 using a matrix
* Creates a deployment archive containing only production-ready files

#### Key Components Explained

**Checkout step**\
Pulls your code into the GitHub Actions runner.

**Node.js setup**\
Automatically installs the required Node version and caches dependencies.

**Installing dependencies**\
`npm ci` ensures clean, reproducible installs.

**Build step**\
Optional. If your app does not have a build step, the workflow continues.

**Testing**\
Runs Jest tests and generates JUnit test reports.

**Publishing test results**\
Displays test summaries inside GitHub Checks.

**Uploading test artifacts**\
Stores coverage reports for later download.

**Creating a deployment package**\
Copies only required files into a staging folder (no dev dependencies).\
Zips the folder with a unique name based on commit hash and timestamp.

**Uploading deployment package**\
Saves it to GitHub for download and later deployment.

***

### Step 5: Push Changes to Trigger the Pipeline

Commit and push your changes:

```bash
git add .
git commit -m "Added CI pipeline"
git push
```

Navigate to the **Actions** tab in your GitHub repository to see the pipeline running.

You will see two jobs (Node 18 and Node 20) executing in parallel.

***

### Step 6: Download and Run the Deployment Artifact

Once the workflow completes:

1. Go to Actions → select the workflow run
2. Download the deployment package zip file
3. Extract it on a machine with Node.js installed
4. Navigate into the extracted folder:

```bash
cd deployment-package
```

5. Start the application:

```bash
npm start
```

6. Access the running application in your browser by navigating to your machine or VM’s IP address.

***

### Completion

You have successfully completed this CI lab and learned how to:

* Create a GitHub Actions workflow
* Run builds and automated tests
* Generate reports and publish test results
* Package and publish application artifacts
* Use matrix builds to test across multiple Node.js versions
