# Lab 1: Your First GitHub Actions Workflow

This lab introduces you to GitHub Actions and guides you through creating your first workflow. You will learn workflow syntax, triggers, job structure, and common best practices. You will then extend your knowledge by working with matrix builds and scheduled workflows.

***

### Prerequisites

* Git installed on your machine
* A text editor such as Visual Studio Code
* A GitHub account with valid credentials

***

### Step 1: Create a New GitHub Repository

1. Log in to your GitHub account.
2. Create a new repository.
3. While creating the repository, enable the option to create a README.md file.
4. Add some text to README.md and commit the file.

This will serve as the initial content for your repository.

***

### Step 2: Add a GitHub Actions Workflow

1. Navigate to the repository on GitHub.
2. Click the **Actions** tab in the top navigation bar.
3. GitHub will display several workflow templates. Select **Simple Workflow** and click **Configure**.

GitHub will open a new editor showing a YAML workflow template inside the folder `.github/workflows`.

Below is the default example you will see.

```
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run a one-line script
        run: echo Hello, world!

      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```

#### Understanding the Workflow

**Workflow name**

```
name: CI
```

This defines the name shown in the Actions tab.

**Triggers**

```
on:
  push:
    branches: [ "main" ]
  pull_request:
  workflow_dispatch:
```

The workflow runs when:

* A commit is pushed to the main branch
* A pull request is opened targeting main
* The workflow is manually triggered from the UI

**Job definition**

```
jobs:
  build:
    runs-on: ubuntu-latest
```

The workflow contains one job named build. It runs on an Ubuntu virtual machine.

**Steps in the build job**

1.  **Checkout code**

    ```
    - uses: actions/checkout@v4
    ```

    This step clones your repository into the runner.
2.  **Run a single command**

    ```
    - name: Run a one-line script
      run: echo Hello, world!
    ```
3.  **Run multiple commands**

    ```
    - name: Run a multi-line script
      run: |
        echo Add other actions to build,
        echo test, and deploy your project.
    ```

#### Save the workflow

Click **Commit changes**, add a commit message, and save.

GitHub will automatically run this workflow since it triggers on pushes to the main branch.

#### View results

1. Go to the **Actions** tab.
2. You will see the workflow run with a green check mark if successful.
3. Click the run to inspect job logs and output.

***

### Step 3: Clone the Repository and Modify the Workflow

Clone the repository:

```bash
git clone <repository-url>
```

Inside the cloned project, notice the `.github/workflows` directory. It contains the CI workflow created earlier.

You will now modify the workflow to run **only on manual triggers** to avoid unnecessary runs.

Replace the workflow contents with:

```
name: CI

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run a one-line script
        run: echo Hello, world!

      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```

Commit and push the changes.

***

### Step 4: More Examples of GitHub Actions

#### Example 1: Matrix Builds

Matrix builds allow you to run the same job across multiple combinations of values such as operating systems or language versions.

Create a new workflow file named `ci-matrix.yml`:

```
name: Matrix Build

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}

      - name: Print environment
        run: |
          node -v
          echo "OS: $RUNNER_OS"
```

Commit and push the file.

#### Run the workflow

1. Go to GitHub → Actions tab.
2. Select **Matrix Build**.
3. Click **Run workflow**.

You will see four jobs run automatically:

* Node 18 on Ubuntu
* Node 20 on Ubuntu
* Node 18 on Windows
* Node 20 on Windows

***

#### Example 2: Scheduled and Manual Workflow

Create another workflow file named `scheduled.yml`:

```
name: Scheduled & Manual

on:
  schedule:
    - cron: '0 0 * * 1'
  workflow_dispatch:
    inputs:
      environment:
        description: Which environment?
        type: choice
        options: [dev, staging, prod]
        default: dev
      message:
        description: Message to print
        required: false
        default: "Hello from manual run"

jobs:
  run-task:
    runs-on: ubuntu-latest
    steps:
      - name: Print context
        run: |
          echo "Event: $GITHUB_EVENT_NAME"
          echo "Env input: ${{ inputs.environment }}"
          echo "Msg: ${{ inputs.message }}"
```

#### What this workflow does

* Runs automatically every Monday at 00:00 UTC
* Can also be triggered manually with user inputs
* Prints the event name, selected environment value, and a message

Commit and push the file, then run it through the Actions tab.

***

## Completion

You have successfully:

1. Created your first GitHub Actions workflow
2. Learned workflow syntax, triggers, jobs, and steps
3. Modified workflows and executed them manually
4. Implemented matrix builds
5. Created scheduled and input-based workflows

You now understand the core concepts needed to begin automating builds, tests, and deployments with GitHub Actions.
