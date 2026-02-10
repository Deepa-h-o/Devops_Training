# GitHub Actions Multistage CI/CD Pipeline

This workflow implements a complete CI/CD pipeline with multiple environments: **Development**, **Staging**, and **Production**.

## Pipeline Stages

### 1. **Build** (Always runs on push)
- Checks out code
- Sets up Docker Buildx
- Logs into container registry
- Builds and pushes Docker image to GitHub Container Registry (GHCR)
- Outputs image tag and digest for downstream jobs

### 2. **Test** (Runs after successful build)
- Sets up Node.js environment
- Installs dependencies
- Runs linting checks
- Executes unit tests
- Uploads coverage reports to Codecov

### 3. **Deploy to Development** (Only on `develop` branch)
- Triggered when code is pushed to `develop` branch
- Configures AWS credentials
- Deploys to dev environment (ECS/K8s)
- **No approval required** for faster iteration

### 4. **Deploy to Staging** (Only on `staging` branch)
- Triggered when code is pushed to `staging` branch
- Runs integration tests before deployment
- Deploys to staging environment
- **Useful for QA testing**

### 5. **Deploy to Production** (Only on `main` branch)
- Triggered when code is pushed to `main` branch
- **Requires manual approval** via GitHub Environments
- Runs smoke tests before deployment
- Performs health checks after deployment
- Sends notifications on completion

### 6. **Notifications** (Always runs after build & test)
- Sends Slack notifications with pipeline status
- Works regardless of success/failure

## Branch Strategy

```
develop  ──(merge)──> staging  ──(merge)──> main
   ↓                      ↓                    ↓
  Dev                  Staging             Production
 (auto)              (+ tests)            (+ approval)
```

## Required Secrets

Add these to your GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Purpose |
|-------------|---------|
| `AWS_ACCESS_KEY_ID` | AWS credentials for dev/staging |
| `AWS_SECRET_ACCESS_KEY` | AWS credentials for dev/staging |
| `AWS_PROD_ACCESS_KEY_ID` | AWS credentials for production |
| `AWS_PROD_SECRET_ACCESS_KEY` | AWS credentials for production |
| `KUBECONFIG_DEV` | Kubernetes config for dev (optional) |
| `KUBECONFIG_STAGING` | Kubernetes config for staging (optional) |
| `KUBECONFIG_PROD` | Kubernetes config for production (optional) |
| `SLACK_WEBHOOK_URL` | Slack webhook for notifications |

## Setting Up GitHub Environments

1. Go to **Settings → Environments**
2. Create three environments:
   - **development** (no approval needed)
   - **staging** (no approval needed)
   - **production** (enable Required reviewers and add team members)

## Customization

### For Node.js Projects
- Already configured with npm commands
- Update test commands in workflow

### For Python Projects
Replace Node setup with:
```yaml
- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
- run: pip install -r requirements.txt
- run: pytest
```

### For Go Projects
```yaml
- uses: actions/setup-go@v4
  with:
    go-version: '1.21'
- run: go test ./...
```

### For Docker or Custom Deployments
- Replace ECS/K8s commands with your deployment script
- Example: `aws ecs update-service` or `helm upgrade`

## Workflow Triggers

- **On push** to `main`, `develop`, `staging` branches
- **On pull requests** to `main`, `develop` branches

Modify the `on:` section to fit your needs (e.g., add `workflow_dispatch` for manual triggers).

## Example: Manual Trigger

Add to the workflow file to allow manual execution:

```yaml
on:
  push:
    branches: [main, develop, staging]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:  # Add this for manual trigger
```

Then, use the **Actions** tab to run the workflow manually.

## Health Checks & Rollback

Add rollback job on production deployment failure:

```yaml
deploy-prod-rollback:
  name: Rollback Production
  runs-on: ubuntu-latest
  needs: deploy-prod
  if: failure()
  steps:
    - name: Revert to previous stable version
      run: |
        echo "Rolling back to previous version..."
        # kubectl rollout undo deployment/app -n prod
```

## Monitoring

- View workflow runs: **Actions** tab in GitHub
- Check job logs: Click on failed job
- Use GitHub branch protection to block merges if checks fail

## Tips

1. **Keep main branch clean**: Only merge tested, approved code
2. **Use pull requests**: Enforce code review before merging
3. **Monitor logs**: Check GitHub Actions logs for deployment issues
4. **Test locally**: Run the same commands locally before pushing
5. **Use artifacts**: Store build outputs with `actions/upload-artifact@v3`

---

**Next Steps:**
1. Copy `.github/workflows/multistage-pipeline.yml` to your repo
2. Add required secrets
3. Set up GitHub Environments
4. Customize deployment commands for your stack
5. Test the workflow with a pull request
