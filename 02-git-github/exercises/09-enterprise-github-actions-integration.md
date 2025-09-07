# Exercise 9: Enterprise GitHub Actions Integration
**Duration: 240 minutes (4 hours)**  
**Difficulty: Advanced**  
**Prerequisites: Exercises 1-8**

## Learning Objectives
By completing this exercise, you will:
- Implement enterprise-grade GitHub Actions workflows
- Configure organization-level security and compliance
- Set up self-hosted runners for specialized workloads
- Implement advanced deployment strategies with approvals
- Create reusable workflow templates for enterprise teams
- Configure comprehensive monitoring and alerting

## Scenario
You're the DevOps lead for a large enterprise implementing GitHub Actions across multiple teams and repositories. You need to establish enterprise governance, security standards, and scalable automation patterns.

## Part 1: Organization Security and Compliance (60 minutes)

### Task 1.1: Organization Policies Setup
Create organization-wide policies and security configurations.

**Instructions:**
1. **Create organization security policy file**
   ```bash
   mkdir -p .github
   cat > .github/SECURITY.md << 'EOF'
   # Security Policy

   ## Supported Versions
   | Version | Supported          |
   | ------- | ------------------ |
   | 2.x.x   | :white_check_mark: |
   | 1.x.x   | :x:                |

   ## Reporting a Vulnerability
   Please report security vulnerabilities to security@company.com
   EOF
   ```

2. **Create dependency review workflow**
   ```yaml
   # .github/workflows/dependency-review.yml
   name: 'Dependency Review'
   on: [pull_request]

   permissions:
     contents: read

   jobs:
     dependency-review:
       runs-on: ubuntu-latest
       steps:
         - name: 'Checkout Repository'
           uses: actions/checkout@v4
         
         - name: 'Dependency Review'
           uses: actions/dependency-review-action@v3
           with:
             fail-on-severity: moderate
             allow-dependencies-licenses: MIT, Apache-2.0, BSD-3-Clause
   ```

3. **Create security scanning workflow**
   ```yaml
   # .github/workflows/security-scan.yml
   name: Security Scan
   on:
     push:
       branches: [main, develop]
     pull_request:
       branches: [main]
     schedule:
       - cron: '0 2 * * 1'

   jobs:
     security-scan:
       runs-on: ubuntu-latest
       permissions:
         contents: read
         security-events: write
       
       steps:
         - uses: actions/checkout@v4
         
         - name: Run Trivy vulnerability scanner
           uses: aquasecurity/trivy-action@master
           with:
             scan-type: 'fs'
             scan-ref: '.'
             format: 'sarif'
             output: 'trivy-results.sarif'
         
         - name: Upload Trivy scan results to GitHub Security tab
           uses: github/codeql-action/upload-sarif@v3
           with:
             sarif_file: 'trivy-results.sarif'
   ```

### Task 1.2: Branch Protection and Required Workflows
Configure comprehensive branch protection with required status checks.

**Instructions:**
1. **Create branch protection workflow**
   ```yaml
   # .github/workflows/branch-protection-check.yml
   name: Branch Protection Compliance
   on:
     push:
       branches: [main, develop, 'release/*']
     pull_request:
       branches: [main, develop]

   jobs:
     compliance-check:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         
         - name: Check commit message format
           run: |
             if ! echo "${{ github.event.head_commit.message }}" | grep -qE '^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}'; then
               echo "❌ Commit message doesn't follow conventional commits format"
               exit 1
             fi
             echo "✅ Commit message format is valid"
         
         - name: Check for required files
           run: |
             required_files=("README.md" "CHANGELOG.md" ".github/CODEOWNERS")
             for file in "${required_files[@]}"; do
               if [[ ! -f "$file" ]]; then
                 echo "❌ Required file $file is missing"
                 exit 1
               fi
             done
             echo "✅ All required files are present"
         
         - name: Validate CODEOWNERS
           run: |
             if ! grep -q "@devops-team" .github/CODEOWNERS; then
               echo "❌ CODEOWNERS must include @devops-team"
               exit 1
             fi
             echo "✅ CODEOWNERS is valid"
   ```

2. **Create CODEOWNERS file**
   ```bash
   cat > .github/CODEOWNERS << 'EOF'
   # Global code owners
   * @devops-team

   # Frontend code owners
   /frontend/ @frontend-team

   # Backend code owners
   /backend/ @backend-team

   # Infrastructure code owners
   /infrastructure/ @infrastructure-team
   /.github/ @devops-team

   # Security sensitive files
   /secrets/ @security-team @devops-team
   Dockerfile* @security-team @devops-team
   docker-compose*.yml @security-team @devops-team
   EOF
   ```

## Part 2: Self-Hosted Runners Configuration (60 minutes)

### Task 2.1: Self-Hosted Runner Setup
Configure and manage self-hosted runners for specialized workloads.

**Instructions:**
1. **Create runner configuration script**
   ```bash
   mkdir -p scripts/runners
   cat > scripts/runners/setup-runner.sh << 'EOF'
   #!/bin/bash
   set -euo pipefail

   # Self-hosted runner setup script
   RUNNER_VERSION="2.311.0"
   RUNNER_USER="github-runner"
   RUNNER_HOME="/home/${RUNNER_USER}"

   # Create runner user
   sudo useradd -m -s /bin/bash ${RUNNER_USER}
   sudo usermod -aG docker ${RUNNER_USER}

   # Download and setup runner
   sudo -u ${RUNNER_USER} bash << 'USEREOF'
   cd ${RUNNER_HOME}
   curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L \
     https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
   tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
   rm actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
   USEREOF

   # Install dependencies
   sudo ${RUNNER_HOME}/bin/installdependencies.sh

   echo "Runner setup completed. Configure with: sudo -u ${RUNNER_USER} ${RUNNER_HOME}/config.sh"
   EOF

   chmod +x scripts/runners/setup-runner.sh
   ```

2. **Create runner monitoring workflow**
   ```yaml
   # .github/workflows/runner-health-check.yml
   name: Runner Health Check
   on:
     schedule:
       - cron: '*/15 * * * *'  # Every 15 minutes
     workflow_dispatch:

   jobs:
     health-check:
       runs-on: self-hosted
       timeout-minutes: 5
       steps:
         - name: Check runner health
           run: |
             echo "🏃 Runner is healthy and responsive"
             echo "Runner name: $RUNNER_NAME"
             echo "Runner OS: $RUNNER_OS"
             echo "Runner architecture: $RUNNER_ARCH"
             
             # Check available disk space
             df -h
             
             # Check available memory
             free -h
             
             # Check Docker daemon status
             docker system info
         
         - name: Notify on failure
           if: failure()
           uses: 8398a7/action-slack@v3
           with:
             status: failure
             channel: '#devops-alerts'
             webhook_url: ${{ secrets.SLACK_WEBHOOK }}
   ```

### Task 2.2: Runner Scaling Configuration
Implement auto-scaling for self-hosted runners.

**Instructions:**
1. **Create runner scaling workflow**
   ```yaml
   # .github/workflows/runner-autoscale.yml
   name: Runner Auto-Scale
   on:
     workflow_run:
       workflows: ["*"]
       types: [queued]
     schedule:
       - cron: '*/5 * * * *'

   jobs:
     scale-runners:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         
         - name: Check queue length
           id: queue
           run: |
             # Simulate checking GitHub API for queue length
             QUEUE_LENGTH=$(( RANDOM % 10 + 1 ))
             echo "queue_length=$QUEUE_LENGTH" >> $GITHUB_OUTPUT
             echo "Current queue length: $QUEUE_LENGTH"
         
         - name: Scale up runners if needed
           if: steps.queue.outputs.queue_length > 5
           run: |
             echo "🚀 Scaling up runners (queue: ${{ steps.queue.outputs.queue_length }})"
             # In real scenario, this would trigger cloud infrastructure scaling
             # aws ec2 run-instances --instance-type t3.medium --count 2
             
         - name: Scale down runners if queue is low
           if: steps.queue.outputs.queue_length < 2
           run: |
             echo "⬇️ Scaling down runners (queue: ${{ steps.queue.outputs.queue_length }})"
             # In real scenario, this would terminate idle runners
   ```

## Part 3: Advanced Deployment Strategies (60 minutes)

### Task 3.1: Blue-Green Deployment with Approvals
Implement blue-green deployment with manual approval gates.

**Instructions:**
1. **Create environment-specific deployment workflow**
   ```yaml
   # .github/workflows/blue-green-deployment.yml
   name: Blue-Green Deployment
   on:
     push:
       branches: [main]
     workflow_dispatch:
       inputs:
         environment:
           description: 'Deployment environment'
           required: true
           default: 'staging'
           type: choice
           options:
           - staging
           - production
         deployment_strategy:
           description: 'Deployment strategy'
           required: true
           default: 'blue-green'
           type: choice
           options:
           - blue-green
           - rolling
           - canary

   jobs:
     build:
       runs-on: ubuntu-latest
       outputs:
         image_tag: ${{ steps.meta.outputs.tags }}
         image_digest: ${{ steps.build.outputs.digest }}
       
       steps:
         - uses: actions/checkout@v4
         
         - name: Set up Docker Buildx
           uses: docker/setup-buildx-action@v3
         
         - name: Login to registry
           uses: docker/login-action@v3
           with:
             registry: ghcr.io
             username: ${{ github.actor }}
             password: ${{ secrets.GITHUB_TOKEN }}
         
         - name: Extract metadata
           id: meta
           uses: docker/metadata-action@v5
           with:
             images: ghcr.io/${{ github.repository }}
             tags: |
               type=ref,event=branch
               type=sha,prefix={{branch}}-
               type=raw,value=latest,enable={{is_default_branch}}
         
         - name: Build and push
           id: build
           uses: docker/build-push-action@v5
           with:
             context: .
             push: true
             tags: ${{ steps.meta.outputs.tags }}
             labels: ${{ steps.meta.outputs.labels }}
             cache-from: type=gha
             cache-to: type=gha,mode=max

     deploy-staging:
       needs: build
       runs-on: ubuntu-latest
       environment: 
         name: staging
         url: https://staging.example.com
       
       steps:
         - name: Deploy to staging (blue slot)
           run: |
             echo "🚀 Deploying to staging blue slot"
             echo "Image: ${{ needs.build.outputs.image_tag }}"
             
             # Simulate blue-green deployment
             echo "1. Deploying to blue environment"
             echo "2. Running health checks"
             echo "3. Switching traffic to blue"
             echo "4. Keeping green as backup"
         
         - name: Run integration tests
           run: |
             echo "🧪 Running integration tests against staging"
             # Simulate test execution
             for test in "api_health" "database_connection" "user_authentication"; do
               echo "✅ $test: PASSED"
               sleep 2
             done
         
         - name: Performance testing
           run: |
             echo "⚡ Running performance tests"
             # Simulate performance testing
             echo "Average response time: 120ms"
             echo "95th percentile: 250ms"
             echo "Error rate: 0.1%"

     deploy-production:
       needs: [build, deploy-staging]
       runs-on: ubuntu-latest
       environment: 
         name: production
         url: https://app.example.com
       if: github.ref == 'refs/heads/main'
       
       steps:
         - name: Pre-deployment approval gate
           uses: trstringer/manual-approval@v1
           with:
             secret: ${{ secrets.GITHUB_TOKEN }}
             approvers: devops-team,security-team
             minimum-approvals: 2
             issue-title: "Production Deployment Approval Required"
             issue-body: |
               ## Production Deployment Request
               
               **Image:** ${{ needs.build.outputs.image_tag }}
               **Digest:** ${{ needs.build.outputs.image_digest }}
               **Strategy:** Blue-Green
               
               ### Pre-deployment Checklist
               - [ ] Security scan passed
               - [ ] Integration tests passed
               - [ ] Performance tests acceptable
               - [ ] Database migrations ready
               - [ ] Rollback plan confirmed
               
               Please review and approve this deployment.
         
         - name: Deploy to production (blue slot)
           run: |
             echo "🚀 Deploying to production blue slot"
             echo "Image: ${{ needs.build.outputs.image_tag }}"
             
             # Simulate blue-green production deployment
             echo "1. Deploying to blue environment"
             echo "2. Running smoke tests"
             echo "3. Gradual traffic switching (10%, 50%, 100%)"
             sleep 10
             echo "4. Production deployment completed"
         
         - name: Post-deployment verification
           run: |
             echo "✅ Post-deployment verification"
             echo "- Health checks: PASSED"
             echo "- Metrics collection: ACTIVE"
             echo "- Error monitoring: ACTIVE"
             echo "- Rollback capability: CONFIRMED"
   ```

### Task 3.2: Canary Deployment with Metrics
Implement canary deployment with automated metrics evaluation.

**Instructions:**
1. **Create canary deployment workflow**
   ```yaml
   # .github/workflows/canary-deployment.yml
   name: Canary Deployment
   on:
     workflow_dispatch:
       inputs:
         canary_percentage:
           description: 'Canary traffic percentage'
           required: true
           default: '10'
           type: choice
           options:
           - '5'
           - '10'
           - '25'
           - '50'

   jobs:
     canary-deploy:
       runs-on: ubuntu-latest
       environment: production
       
       steps:
         - uses: actions/checkout@v4
         
         - name: Deploy canary version
           run: |
             CANARY_PERCENT=${{ github.event.inputs.canary_percentage }}
             echo "🐤 Deploying canary version with ${CANARY_PERCENT}% traffic"
             
             # Simulate canary deployment
             echo "1. Deploying canary pods"
             echo "2. Configuring load balancer for ${CANARY_PERCENT}% traffic"
             echo "3. Starting metrics collection"
         
         - name: Monitor canary metrics
           run: |
             echo "📊 Monitoring canary metrics for 10 minutes"
             
             for minute in {1..10}; do
               # Simulate metrics collection
               ERROR_RATE=$(echo "scale=2; $RANDOM/32767*2" | bc)
               LATENCY=$(echo "scale=0; 50 + $RANDOM/32767*100" | bc)
               
               echo "Minute $minute:"
               echo "  Error rate: ${ERROR_RATE}%"
               echo "  Avg latency: ${LATENCY}ms"
               
               # Check if metrics are within acceptable range
               if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
                 echo "❌ Error rate too high! Rolling back canary"
                 exit 1
               fi
               
               if (( $(echo "$LATENCY > 200" | bc -l) )); then
                 echo "❌ Latency too high! Rolling back canary"
                 exit 1
               fi
               
               sleep 30
             done
             
             echo "✅ Canary metrics are healthy"
         
         - name: Promote canary to full deployment
           run: |
             echo "🚀 Promoting canary to 100% traffic"
             echo "1. Updating load balancer configuration"
             echo "2. Scaling down old version"
             echo "3. Canary promotion completed"
         
         - name: Rollback on failure
           if: failure()
           run: |
             echo "🔄 Rolling back canary deployment"
             echo "1. Reverting load balancer to 0% canary traffic"
             echo "2. Removing canary pods"
             echo "3. Rollback completed"
   ```

## Part 4: Reusable Workflow Templates (60 minutes)

### Task 4.1: Create Organization Workflow Templates
Develop reusable workflow templates for common enterprise patterns.

**Instructions:**
1. **Create reusable security scan workflow**
   ```yaml
   # .github/workflows/reusable-security-scan.yml
   name: Reusable Security Scan
   on:
     workflow_call:
       inputs:
         scan_type:
           required: true
           type: string
           description: 'Type of security scan (sast, dast, secrets)'
         severity_threshold:
           required: false
           type: string
           default: 'medium'
           description: 'Minimum severity to fail the scan'
         target_path:
           required: false
           type: string
           default: '.'
           description: 'Path to scan'
       secrets:
         SECURITY_SCAN_TOKEN:
           required: false
       outputs:
         scan_results:
           description: 'Security scan results'
           value: ${{ jobs.security-scan.outputs.results }}

   jobs:
     security-scan:
       runs-on: ubuntu-latest
       outputs:
         results: ${{ steps.scan.outputs.results }}
       
       steps:
         - uses: actions/checkout@v4
         
         - name: SAST Scan
           if: inputs.scan_type == 'sast'
           id: scan
           run: |
             echo "🔍 Running SAST scan on ${{ inputs.target_path }}"
             
             # Simulate SAST scan results
             cat > scan-results.json << 'EOF'
             {
               "vulnerabilities": [
                 {
                   "severity": "high",
                   "type": "SQL Injection",
                   "file": "src/database/query.js",
                   "line": 42,
                   "description": "Potential SQL injection vulnerability"
                 },
                 {
                   "severity": "medium",
                   "type": "XSS",
                   "file": "src/web/template.js", 
                   "line": 18,
                   "description": "Potential XSS vulnerability"
                 }
               ]
             }
             EOF
             
             # Check if any high severity issues found
             HIGH_COUNT=$(jq '[.vulnerabilities[] | select(.severity == "high")] | length' scan-results.json)
             
             echo "results=$(cat scan-results.json | jq -c .)" >> $GITHUB_OUTPUT
             
             if [[ "$HIGH_COUNT" -gt 0 ]] && [[ "${{ inputs.severity_threshold }}" == "high" ]]; then
               echo "❌ $HIGH_COUNT high severity vulnerabilities found"
               exit 1
             fi
             
             echo "✅ Security scan completed"
         
         - name: Secrets Scan
           if: inputs.scan_type == 'secrets'
           run: |
             echo "🔐 Running secrets scan"
             
             # Check for common secret patterns
             if grep -r -i "password\s*=\s*['\"][^'\"]*['\"]" ${{ inputs.target_path }} || \
                grep -r -i "api[_-]?key\s*=\s*['\"][^'\"]*['\"]" ${{ inputs.target_path }} || \
                grep -r -i "secret\s*=\s*['\"][^'\"]*['\"]" ${{ inputs.target_path }}; then
               echo "❌ Potential secrets found in code"
               exit 1
             fi
             
             echo "✅ No secrets detected"
         
         - name: Upload security results
           if: always()
           uses: actions/upload-artifact@v4
           with:
             name: security-scan-results-${{ inputs.scan_type }}
             path: scan-results.json
             retention-days: 30
   ```

2. **Create reusable deployment workflow**
   ```yaml
   # .github/workflows/reusable-deployment.yml
   name: Reusable Deployment
   on:
     workflow_call:
       inputs:
         environment:
           required: true
           type: string
         image_tag:
           required: true
           type: string
         deployment_strategy:
           required: false
           type: string
           default: 'rolling'
         health_check_url:
           required: false
           type: string
         rollback_on_failure:
           required: false
           type: boolean
           default: true
       secrets:
         DEPLOY_TOKEN:
           required: true
         SLACK_WEBHOOK:
           required: false

   jobs:
     deploy:
       runs-on: ubuntu-latest
       environment: ${{ inputs.environment }}
       
       steps:
         - uses: actions/checkout@v4
         
         - name: Pre-deployment validation
           run: |
             echo "🔍 Validating deployment parameters"
             echo "Environment: ${{ inputs.environment }}"
             echo "Image tag: ${{ inputs.image_tag }}"
             echo "Strategy: ${{ inputs.deployment_strategy }}"
             
             # Validate environment
             if [[ "${{ inputs.environment }}" != "staging" && "${{ inputs.environment }}" != "production" ]]; then
               echo "❌ Invalid environment: ${{ inputs.environment }}"
               exit 1
             fi
             
             echo "✅ Deployment parameters validated"
         
         - name: Rolling deployment
           if: inputs.deployment_strategy == 'rolling'
           run: |
             echo "🚀 Starting rolling deployment"
             echo "Deploying image: ${{ inputs.image_tag }}"
             
             # Simulate rolling deployment
             for replica in {1..3}; do
               echo "Updating replica $replica/3"
               echo "  - Stopping old container"
               echo "  - Starting new container with ${{ inputs.image_tag }}"
               echo "  - Waiting for health check"
               sleep 5
             done
             
             echo "✅ Rolling deployment completed"
         
         - name: Blue-green deployment
           if: inputs.deployment_strategy == 'blue-green'
           run: |
             echo "🔄 Starting blue-green deployment"
             
             # Simulate blue-green deployment
             echo "1. Deploying to green environment"
             echo "2. Running health checks"
             echo "3. Switching load balancer to green"
             echo "4. Blue environment kept as backup"
             
             echo "✅ Blue-green deployment completed"
         
         - name: Health check
           if: inputs.health_check_url != ''
           run: |
             echo "🏥 Running health check: ${{ inputs.health_check_url }}"
             
             for attempt in {1..10}; do
               if curl -f "${{ inputs.health_check_url }}" > /dev/null 2>&1; then
                 echo "✅ Health check passed (attempt $attempt)"
                 break
               else
                 echo "⏳ Health check failed (attempt $attempt), retrying..."
                 if [[ $attempt -eq 10 ]]; then
                   echo "❌ Health check failed after 10 attempts"
                   exit 1
                 fi
                 sleep 30
               fi
             done
         
         - name: Rollback on failure
           if: failure() && inputs.rollback_on_failure
           run: |
             echo "🔄 Rolling back deployment due to failure"
             
             # Simulate rollback
             echo "1. Switching back to previous version"
             echo "2. Scaling up previous deployment"
             echo "3. Removing failed deployment"
             
             echo "✅ Rollback completed"
         
         - name: Notify deployment status
           if: always() && secrets.SLACK_WEBHOOK != ''
           uses: 8398a7/action-slack@v3
           with:
             status: ${{ job.status }}
             channel: '#deployments'
             webhook_url: ${{ secrets.SLACK_WEBHOOK }}
             fields: repo,message,commit,author,action,eventName,ref,workflow
   ```

### Task 4.2: Create Workflow Template Usage Examples
Create examples showing how to use the reusable workflows.

**Instructions:**
1. **Create example application workflow using templates**
   ```yaml
   # .github/workflows/application-pipeline.yml
   name: Application Pipeline
   on:
     push:
       branches: [main, develop]
     pull_request:
       branches: [main]

   jobs:
     security-sast:
       uses: ./.github/workflows/reusable-security-scan.yml
       with:
         scan_type: 'sast'
         severity_threshold: 'high'
         target_path: './src'
       secrets:
         SECURITY_SCAN_TOKEN: ${{ secrets.SECURITY_SCAN_TOKEN }}
     
     security-secrets:
       uses: ./.github/workflows/reusable-security-scan.yml
       with:
         scan_type: 'secrets'
         target_path: '.'
     
     build-and-test:
       runs-on: ubuntu-latest
       needs: [security-sast, security-secrets]
       
       steps:
         - uses: actions/checkout@v4
         
         - name: Setup Node.js
           uses: actions/setup-node@v4
           with:
             node-version: '18'
             cache: 'npm'
         
         - name: Install dependencies
           run: npm ci
         
         - name: Run tests
           run: npm test -- --coverage
         
         - name: Build application
           run: npm run build
         
         - name: Upload build artifacts
           uses: actions/upload-artifact@v4
           with:
             name: build-artifacts
             path: dist/
     
     deploy-staging:
       if: github.ref == 'refs/heads/develop'
       needs: build-and-test
       uses: ./.github/workflows/reusable-deployment.yml
       with:
         environment: 'staging'
         image_tag: 'myapp:staging-${{ github.sha }}'
         deployment_strategy: 'rolling'
         health_check_url: 'https://staging.example.com/health'
       secrets:
         DEPLOY_TOKEN: ${{ secrets.STAGING_DEPLOY_TOKEN }}
         SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
     
     deploy-production:
       if: github.ref == 'refs/heads/main'
       needs: build-and-test
       uses: ./.github/workflows/reusable-deployment.yml
       with:
         environment: 'production'
         image_tag: 'myapp:prod-${{ github.sha }}'
         deployment_strategy: 'blue-green'
         health_check_url: 'https://app.example.com/health'
         rollback_on_failure: true
       secrets:
         DEPLOY_TOKEN: ${{ secrets.PRODUCTION_DEPLOY_TOKEN }}
         SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
   ```

2. **Create comprehensive monitoring workflow**
   ```yaml
   # .github/workflows/monitoring-and-alerting.yml
   name: Monitoring and Alerting
   on:
     schedule:
       - cron: '*/5 * * * *'  # Every 5 minutes
     workflow_dispatch:

   jobs:
     infrastructure-monitoring:
       runs-on: ubuntu-latest
       steps:
         - name: Check service health
           run: |
             echo "🏥 Checking service health"
             
             services=("api.example.com" "app.example.com" "admin.example.com")
             
             for service in "${services[@]}"; do
               if curl -f "https://$service/health" > /dev/null 2>&1; then
                 echo "✅ $service is healthy"
               else
                 echo "❌ $service is unhealthy"
                 # Send alert
                 echo "service=$service" >> unhealthy-services.txt
               fi
             done
         
         - name: Check database connectivity
           run: |
             echo "🗄️ Checking database connectivity"
             
             # Simulate database health checks
             dbs=("primary-db" "replica-db" "cache-db")
             
             for db in "${dbs[@]}"; do
               # Simulate connection check
               if [[ $RANDOM -gt 16383 ]]; then
                 echo "❌ $db connection failed"
                 echo "database=$db" >> unhealthy-services.txt
               else
                 echo "✅ $db connection successful"
               fi
             done
         
         - name: Send alerts for unhealthy services
           if: hashFiles('unhealthy-services.txt') != ''
           uses: 8398a7/action-slack@v3
           with:
             status: failure
             channel: '#critical-alerts'
             webhook_url: ${{ secrets.SLACK_WEBHOOK }}
             fields: |
               Unhealthy Services Detected:
               $(cat unhealthy-services.txt | sed 's/^/- /')
     
     performance-monitoring:
       runs-on: ubuntu-latest
       steps:
         - name: Monitor response times
           run: |
             echo "⚡ Monitoring response times"
             
             endpoints=("/" "/api/health" "/api/users" "/api/orders")
             
             for endpoint in "${endpoints[@]}"; do
               # Simulate response time check
               response_time=$((RANDOM % 500 + 50))
               
               echo "Endpoint $endpoint: ${response_time}ms"
               
               if [[ $response_time -gt 300 ]]; then
                 echo "⚠️ Slow response detected: $endpoint (${response_time}ms)"
                 echo "endpoint=$endpoint,time=${response_time}ms" >> slow-endpoints.txt
               fi
             done
         
         - name: Alert on slow endpoints
           if: hashFiles('slow-endpoints.txt') != ''
           run: |
             echo "📊 Slow endpoints detected:"
             cat slow-endpoints.txt
   ```

## Verification and Testing

### Task 5.1: Test Enterprise Workflows
Test all the enterprise workflows you've created.

**Instructions:**
1. **Test security scanning workflow**
   ```bash
   # Create a test file with potential security issues
   mkdir -p test-app/src
   cat > test-app/src/config.js << 'EOF'
   // This file contains intentional security issues for testing
   const config = {
     database: {
       password: "hardcoded_password_123",
       host: "localhost"
     },
     api_key: "sk-1234567890abcdef",
     secret: "my_super_secret_key"
   };
   
   // Potential SQL injection
   function getUserById(id) {
     return query(`SELECT * FROM users WHERE id = '${id}'`);
   }
   
   module.exports = config;
   EOF
   
   # Test the security scan
   echo "Testing security scan workflow..."
   git add test-app/
   git commit -m "Add test app for security scanning"
   ```

2. **Test deployment workflows**
   ```bash
   # Create a simple health check endpoint
   mkdir -p test-app/public
   cat > test-app/public/health.html << 'EOF'
   <!DOCTYPE html>
   <html>
   <head>
       <title>Health Check</title>
   </head>
   <body>
       <h1>Service is healthy</h1>
       <p>Status: OK</p>
       <p>Timestamp: <span id="timestamp"></span></p>
       <script>
           document.getElementById('timestamp').textContent = new Date().toISOString();
       </script>
   </body>
   </html>
   EOF
   
   # Add to git
   git add test-app/public/
   git commit -m "Add health check endpoint for deployment testing"
   ```

### Task 5.2: Create Enterprise Documentation
Document the enterprise GitHub Actions setup.

**Instructions:**
1. **Create enterprise workflows documentation**
   ```bash
   mkdir -p docs/github-actions
   cat > docs/github-actions/enterprise-setup.md << 'EOF'
   # Enterprise GitHub Actions Setup

   ## Overview
   This document describes the enterprise-grade GitHub Actions configuration implemented for our organization.

   ## Security and Compliance

   ### Branch Protection
   - All main branches require status checks
   - CODEOWNERS file enforces code review requirements
   - Conventional commit messages are enforced
   - Required files must be present (README.md, CHANGELOG.md, etc.)

   ### Security Scanning
   - **SAST scanning**: Runs on every push and PR
   - **Dependency review**: Checks for vulnerable dependencies
   - **Secrets detection**: Prevents secret commits
   - **Container scanning**: Scans Docker images for vulnerabilities

   ## Self-Hosted Runners

   ### Setup
   ```bash
   # Run the setup script
   ./scripts/runners/setup-runner.sh
   
   # Configure runner (replace with actual values)
   sudo -u github-runner /home/github-runner/config.sh \
     --url https://github.com/your-org \
     --token YOUR_RUNNER_TOKEN \
     --name enterprise-runner-01
   ```

   ### Auto-scaling
   - Monitors queue length every 5 minutes
   - Scales up when queue > 5 jobs
   - Scales down when queue < 2 jobs

   ## Deployment Strategies

   ### Blue-Green Deployment
   - Zero-downtime deployments
   - Automatic rollback on failure
   - Manual approval gates for production

   ### Canary Deployment
   - Gradual traffic shifting (5%, 10%, 25%, 50%)
   - Automated metrics monitoring
   - Automatic rollback on metric threshold breach

   ## Reusable Workflows

   ### Security Scan Workflow
   ```yaml
   jobs:
     security:
       uses: ./.github/workflows/reusable-security-scan.yml
       with:
         scan_type: 'sast'
         severity_threshold: 'high'
   ```

   ### Deployment Workflow
   ```yaml
   jobs:
     deploy:
       uses: ./.github/workflows/reusable-deployment.yml
       with:
         environment: 'production'
         image_tag: 'myapp:v1.2.3'
         deployment_strategy: 'blue-green'
   ```

   ## Monitoring and Alerting

   ### Health Monitoring
   - Service health checks every 5 minutes
   - Database connectivity monitoring
   - Automatic Slack alerts for failures

   ### Performance Monitoring
   - Response time tracking
   - Slow endpoint detection
   - Performance trend analysis

   ## Best Practices

   1. **Use reusable workflows** for common patterns
   2. **Implement proper secrets management** with environment-specific secrets
   3. **Use manual approval gates** for production deployments
   4. **Monitor deployment metrics** and set up automatic rollbacks
   5. **Keep workflows modular** and composable
   6. **Use self-hosted runners** for sensitive workloads
   7. **Implement comprehensive security scanning** at multiple stages
   EOF
   ```

2. **Create troubleshooting guide**
   ```bash
   cat > docs/github-actions/troubleshooting.md << 'EOF'
   # GitHub Actions Troubleshooting Guide

   ## Common Issues

   ### Workflow Failures

   #### Security Scan Failures
   ```
   Error: High severity vulnerabilities found
   ```
   **Solution**: Review the security scan results and fix the vulnerabilities before merging.

   #### Deployment Failures
   ```
   Error: Health check failed after 10 attempts
   ```
   **Solution**: 
   1. Check application logs for startup errors
   2. Verify health check endpoint is accessible
   3. Increase health check timeout if needed

   ### Self-Hosted Runner Issues

   #### Runner Not Responding
   ```bash
   # Check runner status
   sudo systemctl status github-runner

   # Restart runner service
   sudo systemctl restart github-runner

   # Check runner logs
   sudo journalctl -u github-runner -f
   ```

   #### Runner Out of Disk Space
   ```bash
   # Clean up Docker resources
   docker system prune -af

   # Clean up runner workspace
   sudo -u github-runner rm -rf /home/github-runner/_work/*
   ```

   ### Deployment Issues

   #### Blue-Green Deployment Stuck
   1. Check load balancer configuration
   2. Verify both blue and green environments are healthy
   3. Manual failover if needed:
   ```bash
   # Switch traffic back to previous version
   kubectl patch service myapp -p '{"spec":{"selector":{"version":"previous"}}}'
   ```

   #### Canary Metrics Failure
   1. Review metrics threshold configuration
   2. Check if baseline metrics are realistic
   3. Adjust canary percentage if needed

   ## Monitoring and Alerts

   ### Slack Notifications Not Working
   1. Verify SLACK_WEBHOOK secret is set
   2. Check webhook URL is still valid
   3. Test webhook manually:
   ```bash
   curl -X POST -H 'Content-type: application/json' \
     --data '{"text":"Test message"}' \
     YOUR_SLACK_WEBHOOK_URL
   ```

   ### Performance Monitoring Issues
   1. Check if monitoring endpoints are accessible
   2. Verify response time thresholds are appropriate
   3. Review monitoring frequency settings

   ## Emergency Procedures

   ### Complete Rollback
   ```bash
   # Stop all deployments
   kubectl scale deployment myapp --replicas=0

   # Deploy last known good version
   kubectl set image deployment/myapp app=myapp:last-good-version

   # Scale back up
   kubectl scale deployment myapp --replicas=3
   ```

   ### Disable Automated Deployments
   1. Go to repository Settings > Environments
   2. Add manual approval requirement for all environments
   3. Temporarily disable workflow triggers if needed
   EOF
   ```

## Exercise Summary

**Congratulations!** You have successfully implemented enterprise-grade GitHub Actions workflows including:

✅ **Organization Security and Compliance**
- Branch protection and required workflows
- Security scanning (SAST, secrets, dependencies)
- Comprehensive compliance checks

✅ **Self-Hosted Runners**
- Runner setup and configuration
- Health monitoring and auto-scaling
- Enterprise-grade runner management

✅ **Advanced Deployment Strategies**
- Blue-green deployment with approval gates
- Canary deployment with metrics monitoring
- Automated rollback capabilities

✅ **Reusable Workflow Templates**
- Security scanning templates
- Deployment workflow templates
- Comprehensive monitoring workflows

✅ **Enterprise Documentation**
- Setup and configuration guides
- Troubleshooting documentation
- Best practices and procedures

### Key Achievements
- **Security**: Multi-layered security scanning and compliance
- **Scalability**: Auto-scaling runner infrastructure
- **Reliability**: Advanced deployment strategies with rollback
- **Maintainability**: Reusable workflow templates
- **Monitoring**: Comprehensive health and performance monitoring

### Next Steps
1. Implement organization-wide security policies
2. Set up centralized logging and metrics collection  
3. Create disaster recovery procedures
4. Establish SLA monitoring and alerting
5. Implement cost optimization strategies

**Total Time Investment: 240 minutes (4 hours)**
**Skills Developed: Enterprise DevOps, GitHub Actions mastery, Security automation, Deployment strategies**