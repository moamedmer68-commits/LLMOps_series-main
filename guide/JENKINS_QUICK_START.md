# Jenkins Quick Start - TL;DR

## The Problem You Had
```
ERROR: azure-tenant-id
Finished: FAILURE
```
**Why:** Pipeline tried to load Azure credentials even for test-only runs.

## The Solution
Two separate pipelines:
1. **`Jenkinsfile.test`** → Tests only (no Azure credentials)
2. **`Jenkinsfile`** → Tests + optional deploy (credentials only when needed)

## Quick Setup (5 minutes)

### 1. Jenkins is Already Running ✅
- URL: **http://localhost:8083**
- Container: `jenkins-local`

### 2. Fix Your Current Job

**In Jenkins UI:**
1. Go to your `run-tests` job
2. Click **Configure**
3. Scroll to **Pipeline** section
4. Change **Script Path** from `Jenkinsfile` to `Jenkinsfile.test`
5. Click **Save**

### 3. Test It

**Option A - Manual:**
Click **Build Now**

**Option B - Automatic:**
Push a commit → Wait 2 minutes → Build auto-triggers

### 4. Verify Success ✅
- Build should complete without credential errors
- Check **Stage View** (Checkout → Setup Python → Install Deps → Run Tests)
- View **Test Result** for test summary
- Download **coverage.xml** or **htmlcov/** artifacts

## That's It! 🎉

No Azure credentials needed. Tests run on every push.

## Commands

```bash
# View Jenkins logs
docker logs -f jenkins-local

# Restart Jenkins
docker compose -f docker-compose.jenkins.yml restart

# Stop Jenkins
docker compose -f docker-compose.jenkins.yml stop

# Start Jenkins
docker compose -f docker-compose.jenkins.yml start
```

## Two Pipelines Comparison

| Feature | Jenkinsfile.test | Jenkinsfile |
|---------|------------------|-------------|
| **Purpose** | Tests only | Tests + Deploy |
| **Credentials** | None (just GitHub PAT for private) | + Azure (when deploying) |
| **Use for** | Local dev, CI | Full CI/CD |
| **Script Path** | `Jenkinsfile.test` | `Jenkinsfile` |

## When to Use Which?

### Use `Jenkinsfile.test`:
- ✅ Local development
- ✅ Running tests on every commit
- ✅ No Azure access needed
- ✅ Quick feedback loop

### Use `Jenkinsfile`:
- ✅ When you need deployment option
- ✅ Production CI/CD pipeline
- ✅ Test first (RUN_DEPLOY=false), deploy later (RUN_DEPLOY=true)
- ⚠️  Requires Azure credentials in Jenkins (only for deploy)

## How Auto-Trigger Works

```
You push code → GitHub
         ↓
Wait ~2 minutes (SCM polling)
         ↓
Jenkins detects change
         ↓
Build auto-starts
         ↓
Tests run + results archived
```

## Need More Details?

Read: **`JENKINS_LOCAL_GUIDE.md`** (comprehensive guide)

## Still Stuck?

### Build still failing with credential error?
→ Make sure Script Path is `Jenkinsfile.test`

### Build not auto-triggering?
→ Wait 2-3 minutes after push, or click "Build Now" once to initialize

### Tests failing?
→ Check Console Output for specific error

### Jenkins not accessible?
→ Check Docker Desktop is running: `docker ps | grep jenkins`

---

**Summary:** Your Jenkins is working. Just change Script Path to `Jenkinsfile.test` and you're done! 🚀

