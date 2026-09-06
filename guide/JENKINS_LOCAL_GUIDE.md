# Local Jenkins Setup Guide

This guide explains how to run Jenkins locally in Docker and set up CI/CD pipelines for testing and deployment.

## Overview

We have two separate Jenkinsfiles for different purposes:

1. **`Jenkinsfile.test`** - Test-only pipeline (no Azure credentials required)
   - Runs on every push via SCM polling
   - Executes unit tests with coverage reports
   - Perfect for local development and testing

2. **`Jenkinsfile`** - Full CI/CD pipeline with optional deployment
   - Runs tests by default (RUN_DEPLOY=false)
   - Optionally deploys to Azure Container Apps (RUN_DEPLOY=true)
   - Requires Azure credentials only when deploying

## Quick Start

### 1. Start Jenkins Locally

```bash
# From repository root
docker compose -f docker-compose.jenkins.yml up -d

# Check logs
docker logs -f jenkins-local
```

Jenkins will be available at: **http://localhost:8083**

### 2. Initial Jenkins Setup

1. **Get initial admin password:**
   ```bash
   docker exec jenkins-local cat /var/jenkins_home/secrets/initialAdminPassword
   ```

2. **Open Jenkins:** http://localhost:8083
3. **Install suggested plugins** (Git, Pipeline, JUnit are pre-installed)
4. **Create admin user**

### 3. Configure GitHub Credentials (if private repo)

1. Go to: **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Kind: **Username with password**
   - Username: your GitHub username
   - Password: your GitHub Personal Access Token (PAT)
   - ID: `github-pat`
   - Description: "GitHub PAT for repo access"

**To create a GitHub PAT:**
- GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
- Generate new token with `repo` scope

### 4. Create Jenkins Jobs

#### Option A: Test-Only Job (Recommended for local development)

1. **New Item** → Name: `llmops-tests` → **Pipeline** → OK
2. **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/yashprogrammer/LLMOps_series.git`
   - Credentials: `github-pat` (if private repo)
   - Branch Specifier: `*/AddingJenkins` (or your branch)
   - Script Path: `Jenkinsfile.test`
3. **Save**

#### Option B: Full Pipeline Job (with optional Azure deployment)

1. **New Item** → Name: `llmops-full-pipeline` → **Pipeline** → OK
2. **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/yashprogrammer/LLMOps_series.git`
   - Credentials: `github-pat` (if private repo)
   - Branch Specifier: `*/AddingJenkins` (or your branch)
   - Script Path: `Jenkinsfile`
3. **Save**

**For Azure deployment (only if using Option B):**
- Add Azure credentials in Jenkins:
  - `azure-client-id` (Secret text)
  - `azure-client-secret` (Secret text)
  - `azure-tenant-id` (Secret text)
  - `azure-subscription-id` (Secret text)
  - `acr-username` (Secret text)
  - `acr-password` (Secret text)

### 5. Run Your First Build

1. Click on your job (e.g., `llmops-tests`)
2. Click **Build Now**
3. Watch the build progress in **Stage View**
4. Check test results under **Test Result**

## Pipeline Details

### Jenkinsfile.test (Test-Only)

**Stages:**
1. ✅ Checkout - Clone repository
2. ✅ Setup Python Environment - Install Python 3.12 with uv
3. ✅ Install Dependencies - Install requirements.txt
4. ✅ Run Tests - Execute pytest with coverage

**Features:**
- No Azure credentials required
- Polls GitHub every ~2 minutes for changes
- Archives test results and coverage reports
- Automatic cleanup of virtual environments

**Trigger:** SCM polling (`H/2 * * * *` - approximately every 2 minutes)

### Jenkinsfile (Full Pipeline)

**Stages:**
1. ✅ Checkout - Clone repository
2. ✅ Setup Python Environment - Install Python 3.12 with uv
3. ✅ Install Dependencies - Install requirements.txt
4. ✅ Run Tests - Execute pytest with coverage
5. 🔒 Login to Azure *(only if RUN_DEPLOY=true)*
6. 🔒 Verify Docker Image Exists *(only if RUN_DEPLOY=true)*
7. 🔒 Deploy to Azure Container Apps *(only if RUN_DEPLOY=true)*
8. 🔒 Verify Deployment *(only if RUN_DEPLOY=true)*

**Parameters:**
- `RUN_DEPLOY` (boolean, default: false) - Enable Azure deployment stages

**To run with deployment:**
1. Click **Build with Parameters**
2. Check ✅ **RUN_DEPLOY**
3. Click **Build**

**Trigger:** SCM polling (`H/2 * * * *` - approximately every 2 minutes)

## How It Works

### SCM Polling

Both pipelines use SCM polling instead of webhooks:
- Jenkins checks GitHub every ~2 minutes for changes
- If changes detected, automatically triggers a build
- No need for ngrok/cloudflare tunnels or GitHub webhook configuration

### Credential Management

**Test-only pipeline (`Jenkinsfile.test`):**
- ✅ No credentials required
- Works out of the box for public repos
- Only needs GitHub PAT for private repos

**Full pipeline (`Jenkinsfile`):**
- ✅ No credentials required for test runs (RUN_DEPLOY=false)
- 🔒 Azure credentials required only when RUN_DEPLOY=true
- Credentials loaded dynamically using `withCredentials`

### Test Results

After each build:
- **Test Result** link shows test pass/fail summary
- **Coverage reports** archived as artifacts (coverage.xml, htmlcov/)
- **JUnit XML** results for trend analysis

## Daily Operations

### Stop Jenkins
```bash
docker compose -f docker-compose.jenkins.yml stop
```

### Start Jenkins
```bash
docker compose -f docker-compose.jenkins.yml start
```

### View Logs
```bash
docker logs -f jenkins-local
```

### Remove Jenkins (including data)
```bash
docker compose -f docker-compose.jenkins.yml down -v
```

### Backup Jenkins Data
```bash
# Jenkins data is stored in Docker volume: jenkins_home
docker run --rm -v jenkins_home:/data -v $(pwd):/backup \
  alpine tar czf /backup/jenkins-backup.tar.gz /data
```

### Restore Jenkins Data
```bash
docker run --rm -v jenkins_home:/data -v $(pwd):/backup \
  alpine tar xzf /backup/jenkins-backup.tar.gz -C /
```

## Troubleshooting

### Build Failing: "Cannot connect to Docker daemon"
- Ensure Docker Desktop is running
- Check that `/var/run/docker.sock` is mounted in `docker-compose.jenkins.yml`

### Build Failing: "azure-tenant-id credential not found"
- This means you're using `Jenkinsfile` with RUN_DEPLOY=true
- Either add Azure credentials or use `Jenkinsfile.test` instead
- Or set RUN_DEPLOY=false when building

### SCM Polling Not Working
- Check Jenkins system log: Manage Jenkins → System Log
- Verify repository URL is correct
- Check credentials if private repo
- Manually trigger one build to initialize workspace

### Python Version Issues
- Pipeline uses Python 3.12 (matches `pyproject.toml`)
- `uv` package manager handles version installation
- Check stage logs for Python version confirmation

### Test Failures
- Check **Console Output** for detailed pytest output
- View **Test Result** for test summary
- Download `htmlcov` artifacts for coverage report

### Port 8083 Already in Use
Edit `docker-compose.jenkins.yml`:
```yaml
ports:
  - "8084:8080"  # Change 8083 to any free port
```

Then restart:
```bash
docker compose -f docker-compose.jenkins.yml down
docker compose -f docker-compose.jenkins.yml up -d
```

## Advanced Configuration

### Multibranch Pipeline

For automatic per-branch pipelines:

1. **New Item** → **Multibranch Pipeline** → Name: `llmops-multibranch`
2. **Branch Sources** → **Add source** → **Git**
   - Project Repository: `https://github.com/yashprogrammer/LLMOps_series.git`
   - Credentials: `github-pat`
3. **Build Configuration**:
   - Mode: **by Jenkinsfile**
   - Script Path: `Jenkinsfile.test`
4. **Scan Multibranch Pipeline Triggers**:
   - ✅ Periodically if not otherwise run
   - Interval: `5 minutes`
5. **Save**

### Environment Variables

Add environment variables in Jenkins job configuration:
- **Pipeline** → **Pipeline** section → **Environment variables**

Or set globally:
- **Manage Jenkins** → **System** → **Global properties** → **Environment variables**

Example:
- Name: `GROQ_API_KEY`
- Value: `your-api-key`

### Notification on Build Failure

Add to `post` section of Jenkinsfile:
```groovy
post {
    failure {
        emailext (
            subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Check console output at ${env.BUILD_URL}",
            to: "your-email@example.com"
        )
    }
}
```

## Comparison: Test-Only vs Full Pipeline

| Feature | Jenkinsfile.test | Jenkinsfile |
|---------|------------------|-------------|
| Checkout & Test | ✅ | ✅ |
| Azure Deployment | ❌ | ✅ (optional) |
| Credentials Required | GitHub PAT only (if private) | + Azure (if deploying) |
| Use Case | Local development, CI testing | Full CI/CD with deployment |
| Setup Complexity | Simple | Moderate |
| Build Time | ~2-5 minutes | ~2-15 minutes (with deploy) |

## Recommended Workflow

### For Local Development:
1. Use **`Jenkinsfile.test`** job
2. Push commits to your branch
3. Jenkins auto-detects and runs tests
4. Fix any failures before merging

### For Production Deployment:
1. Use **`Jenkinsfile`** job
2. Run with RUN_DEPLOY=false for tests
3. After tests pass, run with RUN_DEPLOY=true
4. Monitor deployment stages

## Next Steps

- ✅ Jenkins is running locally at http://localhost:8083
- ✅ Create a test-only job using `Jenkinsfile.test`
- ✅ Push a commit and verify tests run automatically
- ✅ View test results and coverage reports
- ⬜ (Optional) Add Azure credentials for deployment
- ⬜ (Optional) Create full pipeline job using `Jenkinsfile`

## Resources

- Jenkins Documentation: https://www.jenkins.io/doc/
- Pipeline Syntax: https://www.jenkins.io/doc/book/pipeline/syntax/
- Docker Compose: https://docs.docker.com/compose/

## Support

If you encounter issues:
1. Check **Console Output** in Jenkins
2. View **docker logs jenkins-local**
3. Review this guide's Troubleshooting section
4. Check Jenkins system log: Manage Jenkins → System Log

