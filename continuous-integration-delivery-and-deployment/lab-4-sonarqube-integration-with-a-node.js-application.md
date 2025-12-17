# Lab 4: SonarQube Integration with a Node.js Application

In this lab, you will create a continuous integration pipeline that builds, tests, and scans a Node.js application using SonarQube. You will also publish test results and deployment artifacts using GitHub Actions.

***

### Pre-Requisite

* A GitHub account
* SonarQube set up and running (from the previous lab)
* Basic understanding of Node.js and GitHub Actions

***

### Learning Objectives

1. Create a GitHub Actions workflow for a Node.js CI process
2. Integrate SonarQube code scanning into the workflow
3. Publish test results and deployment artifacts
4. Use GitHub repository secrets to securely store SonarQube credentials
5. Understand how CI, testing, packaging, and code analysis work together

***

### Step 1: Prepare the Node.js Application

1. Fork the sample Node.js application repository provided by your instructor.
2. Clone the repository to your machine:

```bash
git clone <repository-url>
cd <repository-folder>
```

3. Create the workflow directory:

```
.github/workflows
```

4. Create a new workflow file:

```
ci-sonar.yml
```

***

### Step 2: Add the CI Workflow Code

Copy the following YAML configuration into `ci-sonar.yml`.

```
name: Continuous Integration Pipeline with SonarQube

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
    environment: dev

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18.x
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
          name: test-results-node
          path: coverage/
          retention-days: 30

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v6.0.0
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        with:
          args: >
            -Dsonar.projectKey=sample-node-app-saurabh
            -Dsonar.organization=sda
            -Dsonar.token=${{ secrets.SONAR_TOKEN }}
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }}
            -Dsonar.sources=.
            -Dsonar.exclusions=node_modules/**,coverage/**,tests/**,**/*.test.js,**/*.spec.js
            -Dsonar.tests=tests/
            -Dsonar.test.inclusions=**/*.test.js,**/*.spec.js
            -Dsonar.coverage.exclusions=node_modules/**,coverage/**,tests/**,**/*.test.js,**/*.spec.js

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
          name: deployment-package-node-${{ github.run_number }}
          path: ${{ env.ZIP_NAME }}
          retention-days: 90
```

***

### Step 3: Understand Key Workflow Sections

#### Workflow name

Defines the pipeline name visible in GitHub Actions.

#### Permissions

Allows the workflow to read code, publish test results, and upload artifacts.

#### Node.js setup

Installs Node.js 18.x and enables npm caching for faster runs.

#### Install dependencies

Uses `npm ci`, which ensures reproducible builds.

#### Build step

Runs `npm run build` but does not fail the workflow if the script is missing.

#### Test execution

Runs `npm run test:ci`.\
Outputs coverage reports and a JUnit XML file.

#### SonarQube scan

Uses the SonarQube GitHub Action to analyze code quality.\
The scan requires two secrets:

* SONAR\_HOST\_URL
* SONAR\_TOKEN

#### Deployment package creation

Copies only required files for deployment into a staging folder.\
Removes dev dependencies (no npm install is performed).\
Creates a timestamped and commit-based zip file.

#### Artifact upload

Both test reports and the deployment zip file are uploaded to GitHub.

***

### Step 4: Add SonarQube Secrets in GitHub

1. Go to your GitHub repository
2. Select **Settings**
3. Select **Secrets and variables**
4. Click **New environment** and name it `dev`
5. Inside the environment, add the secrets:

* SONAR\_HOST\_URL
* SONAR\_TOKEN

#### Generating the SonarQube token

1. Log into SonarQube
2. Open User Icon
3. Select **My Account**
4. Navigate to **Security**
5. Create a new token:
   * Token type: Global Analysis Token
   * Expiration: 30 days

Copy the token immediately. It cannot be viewed later.

***

### Step 5: Run the CI Pipeline

1. Commit and push your changes:

```bash
git add .
git commit -m "Added SonarQube CI workflow"
git push
```

2. Go to the **Actions** tab in GitHub.
3. Select **Continuous Integration Pipeline with SonarQube**.
4. Click **Run workflow**.

The pipeline will execute all steps including testing, scanning, and artifact creation.

***

### Step 6: Review SonarQube Report

1. Log in to the SonarQube dashboard.
2. Go to **Projects**.
3. Locate the project key defined in the workflow:\
   `sample-node-app-saurabh`
4. Open the project to view:

* Bugs
* Vulnerabilities
* Code smells
* Security hotspots
* Coverage
* Duplications

Explore the different tabs and understand the quality of your Node.js application.

***

### Completion

You have successfully completed the lab by:

1. Building and testing a Node.js application
2. Integrating SonarQube into a GitHub Actions CI workflow
3. Publishing test results and coverage
4. Generating and uploading deployment artifacts
5. Viewing scan results within SonarQube
