# Session 3 - GitHub Actions
## Cloud-Native CI/CD: From First Workflow to Production Deployment

---

## Prerequisites

> **Before this session:** You should have the Flask app in a GitHub repo, AWS credentials configured, and familiarity with the CI/CD pipeline flow (test -> build -> scan -> push -> deploy) from Jenkins.

### Required Setup

| Requirement | Details |
|-------------|---------|
| GitHub repo | `cicd-lab-app` with Flask app, Dockerfile, and tests |
| AWS CLI v2 | Configured with `aws configure` |
| AWS Resources | ECS cluster, ECR repo, ALB (same as Session 2) |

---

## ═══════════════════════════════════════════
## Lab 1 - Your First GitHub Actions Workflow
## ═══════════════════════════════════════════

**Objective:** Create your first GitHub Actions workflow, understand the YAML syntax, and set up the same test pipeline you built in Jenkins - now as GHA.

### What You'll Learn
- Workflow YAML structure
- Events, jobs, steps
- `uses:` (actions) vs `run:` (shell commands)
- GitHub-hosted runners
- Matrix testing
- Viewing workflow runs in the GitHub UI

---

### Step 1: Create Your First Workflow

Take the clone of your repo where you have kept your Flask app code:

```bash
mkdir -p .github/workflows
```

**`.github/workflows/ci.yml`:**

```yaml
name: CI - Test and Build

on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'              # Cache pip downloads automatically!

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run pytest
        run: |
          python -m pytest tests/ -v --tb=short --junitxml=test-results.xml

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()              # Upload even if tests fail
        with:
          name: test-results
          path: test-results.xml

  build-image:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test              # Only runs if 'test' job passes

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Get Git metadata
        id: meta
        run: |
          echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          echo "run_number=${{ github.run_number }}" >> $GITHUB_OUTPUT

      - name: Build Docker image
        run: |
          docker build \
            --label "git.commit=${{ steps.meta.outputs.sha_short }}" \
            -t cicd-lab-app:${{ steps.meta.outputs.run_number }}-${{ steps.meta.outputs.sha_short }} \
            -t cicd-lab-app:latest \
            .

      - name: Smoke test container
        run: |
          CONTAINER_ID=$(docker run -d -p 9090:8080 cicd-lab-app:latest)
          sleep 5
          HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9090/health)
          docker stop $CONTAINER_ID
          echo "Health check: HTTP $HTTP_CODE"
          [ "$HTTP_CODE" = "200" ] || exit 1

      - name: Image build summary
        run: |
          echo "## Docker Image Built ✅" >> $GITHUB_STEP_SUMMARY
          echo "**Tag:** \`${{ steps.meta.outputs.run_number }}-${{ steps.meta.outputs.sha_short }}\`" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** \`${{ github.sha }}\`" >> $GITHUB_STEP_SUMMARY
```

### Step 2: Push and Observe

```bash
git add .github/
git commit -m "ci: add GitHub Actions workflow"
git push origin main
```

Go to your GitHub repo -> **Actions** tab. You should see the workflow running.

Explore:
- Click on the workflow run -> see jobs
- Click on a job -> see individual steps
- Expand a step -> see command output
- Check the **Summary** tab for the Docker image build summary

### Step 3: Add Matrix Testing

Update `ci.yml` to add a matrix testing job:

```yaml
  test-matrix:
    name: Test Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
      fail-fast: false      # Run all versions even if one fails

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: python -m pytest tests/ -v
```

Push this change and observe 3 parallel test jobs running simultaneously.

### Step 4: Add Environment Variables and Secrets

1. Go to your GitHub repo -> **Settings -> Secrets and variables -> Actions**
2. Add a **New repository secret**: `SLACK_WEBHOOK_URL` (can be a dummy value for now)

Update `ci.yml` to use it:

```yaml
      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-type: application/json' \
            --data '{"text":"❌ CI failed on ${{ github.repository }} branch ${{ github.ref_name }}"}'
```

Notice: secrets are never shown in logs - they appear as `***`.

### ✅ Lab 1 Success Criteria
- Workflow file committed and running in GitHub Actions
- `test` and `build-image` jobs shown as separate jobs
- `needs: test` dependency enforced
- Matrix testing shows 3 Python versions in parallel
- Artifacts uploaded and downloadable from workflow run

---