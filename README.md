# CI/CD Pipeline for Test Automation

[![CI/CD](https://img.shields.io/badge/CI%2FCD-Active-success.svg)](https://github.com/JasonTeixeira/CI-CD-Pipeline)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-enabled-blue.svg)](https://github.com/features/actions)
[![Jenkins](https://img.shields.io/badge/Jenkins-ready-orange.svg)](https://www.jenkins.io/)

> Complete CI/CD pipeline that ties all my testing frameworks together

I built this to show how automated testing fits into a real deployment pipeline. It's one thing to write tests - it's another to run them automatically on every code change, catch issues before they reach production, and deploy with confidence.

---

## What's CI/CD?

**CI (Continuous Integration):** Every time someone pushes code, tests run automatically. If they fail, you know immediately.

**CD (Continuous Delivery):** Code that passes all tests gets automatically deployed to staging, then (with approval) to production.

**Why it matters:** Catches bugs early, deploys faster, reduces manual work, increases confidence.

---

## What This Pipeline Does

```
Code Push → Tests → Security Scan → Build Docker → Deploy Staging → Smoke Tests
```

**Stages:**

1. **Lint & Format** - Code quality checks (flake8, black)
2. **Unit Tests** - Fast tests run first  
3. **API Tests** - Test endpoints with real database
4. **E2E Tests** - Full user flows in headless browser
5. **Security Scan** - Check for vulnerabilities
6. **Performance Tests** - Load testing (main branch only)
7. **Docker Build** - Package application
8. **Deploy Staging** - Auto-deploy to test environment
9. **Smoke Tests** - Verify staging deployment works

If anything fails, the pipeline stops. Nothing broken reaches production.

---

## Quick Start

### GitHub Actions (Easiest)

Already configured! Just push to GitHub:

```bash
git push origin main
```

Check the Actions tab to see your pipeline running.

### Jenkins (More Control)

1. **Install Jenkins**
```bash
# Using Docker
docker run -d -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```

2. **Setup**
- Open http://localhost:8080
- Install suggested plugins
- Create a pipeline job
- Point it to this repo's Jenkinsfile

3. **Configure Credentials**
```
Docker Hub → Add credentials
Slack (optional) → Add webhook URL
Kubernetes (if deploying) → Add kubeconfig
```

4. **Run**
Jenkins will now run the pipeline on every commit.

---

## Pipeline Details

### GitHub Actions Workflow

Located in `.github/workflows/test-pipeline.yml`

**Triggers:**
- Push to `main` or `develop` branches
- Pull requests to `main`
- Nightly at 2 AM (catches flaky tests)

**Jobs Run in Parallel:**
- Unit tests (Python 3.9, 3.10, 3.11)
- API tests (with PostgreSQL service)
- E2E tests (with headless Chrome)
- Security scans (Bandit, Safety)

**On Main Branch Only:**
- Performance tests
- Docker build & push
- Deploy to staging
- Smoke tests

**Example run time:** 8-12 minutes for full pipeline

### Jenkins Pipeline

Located in `Jenkinsfile`

**Parallel Stages:**
- Linting runs alongside formatting checks
- Security scans run in parallel

**Smart Caching:**
- Caches pip dependencies
- Uses Docker layer caching

**Notifications:**
- Slack on success/failure
- Email on broken builds

---

## Testing Stages Explained

### Why This Order?

**1. Unit Tests First**
- Fastest feedback (seconds)
- Catch basic bugs immediately
- No external dependencies

**2. Integration Tests**
- Test with real database
- Verify components work together
- Still relatively fast (1-2 minutes)

**3. E2E Tests**
- Slowest but most comprehensive
- Test like a real user
- Catch UI/UX issues

**4. Security & Performance**
- Run in parallel with E2E
- Only block deployment if critical issues found

This ordering gives fast feedback on most common issues while catching everything eventually.

---

## Docker Integration

### Multi-Stage Build

The Dockerfile uses multi-stage builds:

```dockerfile
# Builder stage - install dependencies
FROM python:3.10-slim as builder
# ... install everything

# Final stage - copy only what's needed
FROM python:3.10-slim
COPY --from=builder ...
```

**Benefits:**
- Smaller final image (200MB vs 800MB)
- Faster deployments
- Better security (fewer attack surfaces)

### Running Tests in Docker

```bash
# Build test image
docker build -t test-app:test .

# Run tests
docker run test-app:test pytest

# Run with coverage
docker run test-app:test pytest --cov=.
```

---

## Deployment Strategy

### Staging First

Every commit to `main` automatically deploys to staging:

```
main branch → tests pass → build Docker → deploy staging → smoke tests
```

Staging is identical to production but with test data. Catches environment-specific issues.

### Production Deployment

Production needs manual approval:

```
Staging tests pass → Manual approval → Deploy production → Monitor
```

This prevents accidental production deployments and gives time to verify staging.

### Rollback Plan

If production deploy fails:

```bash
# Revert to previous version
kubectl rollout undo deployment/app

# Or deploy specific version
kubectl set image deployment/app app=myapp:v1.2.3
```

---

## Environment Variables

Set these in your CI/CD system:

**Required:**
```
DATABASE_URL          # Connection string
SECRET_KEY            # App secret
```

**For Docker:**
```
DOCKERHUB_USERNAME    # Docker Hub user
DOCKERHUB_TOKEN       # Docker Hub token
```

**For Deployment:**
```
KUBECONFIG            # Kubernetes config
STAGING_URL           # Staging environment
PRODUCTION_URL        # Production environment
```

**For Notifications:**
```
SLACK_WEBHOOK         # Slack webhook URL
```

---

## Common Issues

### "Tests pass locally but fail in CI"

**Causes:**
- Different Python versions
- Missing environment variables
- Different database state
- Timing issues (tests run faster/slower)

**Solutions:**
```bash
# Test with same Python version as CI
pyenv install 3.10

# Run tests in Docker (matches CI environment)
docker-compose up test

# Check for race conditions
pytest tests/ --count=10  # Run 10 times
```

### "Docker build is slow"

**Solutions:**
- Use layer caching
- Order Dockerfile properly (dependencies before code)
- Use multi-stage builds
- Cache pip packages

```dockerfile
# Good - dependencies cached
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .  # Code changes don't rebuild dependencies

# Bad - rebuilds everything on code change
COPY . .
RUN pip install -r requirements.txt
```

### "Pipeline takes forever"

**Optimizations:**
- Run tests in parallel
- Cache dependencies
- Use matrix strategy for multiple versions
- Skip unnecessary stages on feature branches

```yaml
# GitHub Actions - run on 3 Python versions in parallel
strategy:
  matrix:
    python-version: ['3.9', '3.10', '3.11']
```

---

## What I Learned

Building this taught me:

**About CI/CD:**
- Fast feedback matters more than perfect coverage
- Parallel execution saves a lot of time
- Caching is crucial for reasonable build times
- Notifications need to be actionable, not just noise

**About Testing in CI:**
- Flaky tests kill trust in the pipeline
- Headless browsers are finicky
- Database state management is harder than it looks
- Timeouts need to be generous in CI environments

**About Docker:**
- Layer order matters for caching
- Multi-stage builds dramatically reduce image size
- Health checks are important for deployments
- Build context size affects speed

**About Deployment:**
- Always deploy to staging first
- Smoke tests after deployment catch configuration issues
- Rollback needs to be easy and tested
- Monitoring is part of the deployment process

---

## Tips for Your Pipeline

### Do:
✅ Start simple - get basic tests running first
✅ Fail fast - run quick tests before slow ones
✅ Cache aggressively - saves minutes per build
✅ Test the pipeline itself - run it on a schedule
✅ Make failures obvious - good notifications matter

### Don't:
❌ Run everything on every branch - it's slow and expensive
❌ Deploy without smoke tests - catches issues early
❌ Ignore flaky tests - fix or remove them
❌ Make the pipeline too complex - simple is maintainable
❌ Skip rollback testing - you'll need it eventually

---

## Project Structure

```
CI-CD-Pipeline/
├── .github/
│   └── workflows/
│       └── test-pipeline.yml    # GitHub Actions config
├── jenkins/
│   └── Jenkinsfile              # Jenkins pipeline
├── docker/
│   ├── Dockerfile               # Main app image
│   └── docker-compose.test.yml  # Test environment
├── scripts/
│   ├── deploy.sh               # Deployment script
│   └── smoke-tests.sh          # Post-deploy checks
└── config/
    ├── staging.env             # Staging config
    └── production.env          # Production config
```

---

## Integrating Your Tests

This pipeline is designed to work with:

**E2E Tests** (Selenium/Playwright)
```yaml
- name: Run E2E tests
  run: pytest tests/e2e --headless
```

**API Tests** (requests/httpx)
```yaml
- name: Run API tests
  run: pytest tests/api
```

**Performance Tests** (Locust)
```yaml
- name: Load test
  run: locust --headless --users 100
```

**Security Tests** (Bandit/Safety)
```yaml
- name: Security scan
  run: bandit -r . && safety check
```

---

## Monitoring & Alerts

### What to Monitor

**Build Health:**
- Success/failure rate
- Build duration trends
- Flaky test frequency

**Deployment:**
- Deployment frequency
- Lead time (commit to production)
- Mean time to recovery (MTTR)

**Tests:**
- Test execution time
- Coverage trends
- Failure patterns

### Tools

- **GitHub Actions**: Built-in insights
- **Jenkins**: Blue Ocean plugin
- **Grafana**: Custom dashboards
- **Slack/Email**: Failure notifications

---

## Cost Considerations

**GitHub Actions:**
- 2,000 free minutes/month for private repos
- ~$0.008/minute after that
- Public repos: unlimited free

**Jenkins (Self-Hosted):**
- Server costs (~$50-200/month depending on size)
- No per-minute charges
- More control but more maintenance

**My Recommendation:** Start with GitHub Actions (it's simpler), move to Jenkins if you need more control or hit usage limits.

---

## Contributing

Found improvements? Open an issue or PR!

---

## Author

**Jason Teixeira**
- GitHub: [@JasonTeixeira](https://github.com/JasonTeixeira)
- Email: sage@sageideas.org

---

## License

MIT License - use it however you want.

---

## Why This Matters

I built this because:
- **Tests are only useful if they run** - CI/CD makes them automatic
- **Manual deployments are error-prone** - automation reduces mistakes  
- **Fast feedback enables fast development** - know immediately if you broke something
- **Confidence to deploy** - comprehensive pipeline means less fear of releasing

A good CI/CD pipeline is the difference between deploying weekly with anxiety and deploying daily with confidence.
