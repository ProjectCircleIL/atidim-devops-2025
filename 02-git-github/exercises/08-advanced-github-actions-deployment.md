# Exercise 8: Advanced GitHub Actions - Deployment Automation & GitOps

**Duration:** 240 minutes  
**Difficulty:** Advanced  
**Type:** Production Deployment + GitOps Implementation

## Objective
Implement advanced GitHub Actions workflows for automated deployment, environment management, and GitOps practices. Build production-ready deployment pipelines with approval workflows, environment promotion, and rollback capabilities.

## Prerequisites
- Completed Exercise 7 (GitHub Actions Fundamentals)
- Working CI pipeline with containerized applications
- Understanding of deployment strategies and environment management
- Access to cloud provider (AWS/Azure/GCP) for deployment targets

## Advanced Deployment Concepts

### Topics to Master:
1. **Environment-Specific Deployments**: Dev, Staging, Production workflows
2. **Approval Workflows**: Manual gates and review processes
3. **Deployment Strategies**: Blue-Green, Canary, Rolling deployments
4. **GitOps Integration**: Automated configuration updates
5. **Rollback Automation**: Failure detection and automatic rollbacks
6. **Multi-Cloud Deployments**: Platform-agnostic deployment patterns
7. **Infrastructure Integration**: Terraform and cloud resource management
8. **Monitoring Integration**: Deployment health monitoring and alerting

## Tasks

### Task 1: Environment Management and Approval Workflows (60 minutes)

**1.1 Environment Configuration with Protection Rules**

Set up GitHub Environments with protection rules:

```yaml
# File: .github/workflows/04-environment-management.yml
name: Environment Management & Deployment

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target Environment'
        required: true
        default: 'development'
        type: choice
        options:
        - development
        - staging
        - production
      force_deploy:
        description: 'Force deployment (skip checks)'
        required: false
        type: boolean
        default: false

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: devops-learning-app

jobs:
  # Job 1: Deployment to Development Environment
  deploy-development:
    name: Deploy to Development
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.example.com
    
    if: github.ref == 'refs/heads/develop' || (github.event.inputs.environment == 'development')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      # Deploy to development ECS cluster
      - name: Deploy to ECS Development
        run: |
          # Update ECS service with new image
          aws ecs update-service \
            --cluster development-cluster \
            --service devops-app-service \
            --task-definition devops-app-dev:REVISION \
            --force-new-deployment
          
          # Wait for deployment to complete
          aws ecs wait services-stable \
            --cluster development-cluster \
            --services devops-app-service
      
      - name: Run smoke tests
        run: |
          echo "Running smoke tests against development environment..."
          curl -f https://dev.example.com/health || exit 1
          curl -f https://dev.example.com/api/status || exit 1
          echo "Development smoke tests passed!"
      
      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: success
          channel: '#deployments'
          message: |
            ✅ Development deployment successful!
            Environment: development
            Commit: ${{ github.sha }}
            Actor: ${{ github.actor }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # Job 2: Deployment to Staging with Approval
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [deploy-development]
    environment:
      name: staging
      url: https://staging.example.com
    
    # Only deploy to staging from main branch or manual trigger
    if: github.ref == 'refs/heads/main' || (github.event.inputs.environment == 'staging')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Wait for approval
        uses: trstringer/manual-approval@v1
        if: github.event.inputs.force_deploy != 'true'
        with:
          secret: ${{ github.TOKEN }}
          approvers: devops-team,platform-team
          minimum-approvals: 2
          issue-title: "Staging Deployment Approval Required"
          issue-body: |
            ## Staging Deployment Request
            
            **Branch**: ${{ github.ref_name }}
            **Commit**: ${{ github.sha }}
            **Actor**: ${{ github.actor }}
            **Environment**: staging
            
            ### Changes in this deployment:
            ${{ github.event.head_commit.message }}
            
            ### Pre-deployment checklist:
            - [ ] All tests passing
            - [ ] Security scans completed
            - [ ] Performance tests acceptable
            - [ ] Database migrations reviewed
            - [ ] Rollback plan confirmed
            
            **Please review and approve this deployment.**
          exclude-workflow-initiator-as-approver: false
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      # Database migration (if needed)
      - name: Run database migrations
        run: |
          echo "Running database migrations for staging..."
          # Run migrations against staging database
          # kubectl exec -it migration-job -- npm run db:migrate
          echo "Database migrations completed"
      
      # Blue-Green deployment to staging
      - name: Deploy to Staging (Blue-Green)
        id: deploy
        run: |
          echo "Starting Blue-Green deployment to staging..."
          
          # Deploy to blue environment first
          aws ecs update-service \
            --cluster staging-cluster \
            --service devops-app-blue \
            --task-definition devops-app-staging:REVISION \
            --force-new-deployment
          
          # Wait for blue deployment to be stable
          aws ecs wait services-stable \
            --cluster staging-cluster \
            --services devops-app-blue
          
          echo "Blue environment deployed successfully"
          echo "blue_url=https://blue-staging.example.com" >> $GITHUB_OUTPUT
      
      - name: Run comprehensive tests against blue environment
        run: |
          echo "Running comprehensive tests against blue environment..."
          
          # Health checks
          curl -f ${{ steps.deploy.outputs.blue_url }}/health
          
          # API tests
          npm run test:api -- --baseURL=${{ steps.deploy.outputs.blue_url }}
          
          # UI tests
          npm run test:e2e -- --baseURL=${{ steps.deploy.outputs.blue_url }}
          
          echo "All tests passed on blue environment"
      
      - name: Switch traffic to blue environment
        run: |
          echo "Switching traffic to blue environment..."
          
          # Update load balancer to point to blue
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.STAGING_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ secrets.BLUE_TARGET_GROUP_ARN }}
          
          # Wait for traffic switch
          sleep 30
          
          echo "Traffic switched to blue environment"
      
      - name: Monitor deployment health
        run: |
          echo "Monitoring deployment health for 5 minutes..."
          
          for i in {1..10}; do
            if curl -f https://staging.example.com/health; then
              echo "Health check $i/10 passed"
            else
              echo "Health check $i/10 failed"
              exit 1
            fi
            sleep 30
          done
          
          echo "Staging deployment is healthy!"

  # Job 3: Production Deployment with Advanced Strategies
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    environment:
      name: production
      url: https://example.com
    
    # Only deploy to production from main branch or manual trigger
    if: github.ref == 'refs/heads/main' || (github.event.inputs.environment == 'production')
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Production deployment requires multiple approvals
      - name: Wait for production approval
        uses: trstringer/manual-approval@v1
        if: github.event.inputs.force_deploy != 'true'
        with:
          secret: ${{ github.TOKEN }}
          approvers: platform-leads,security-team,product-owners
          minimum-approvals: 3
          issue-title: "🚀 PRODUCTION Deployment Approval Required"
          issue-body: |
            ## 🚨 PRODUCTION DEPLOYMENT REQUEST 🚨
            
            **Branch**: ${{ github.ref_name }}
            **Commit**: ${{ github.sha }}
            **Actor**: ${{ github.actor }}
            **Environment**: PRODUCTION
            **Staging URL**: https://staging.example.com
            
            ### 📋 Pre-Production Checklist:
            - [ ] Staging deployment successful and tested
            - [ ] Performance benchmarks met
            - [ ] Security scans passed
            - [ ] Database backup completed
            - [ ] Rollback plan documented
            - [ ] Monitoring alerts configured
            - [ ] On-call team notified
            
            ### 🔄 Rollback Information:
            - **Previous Version**: ${{ github.event.before }}
            - **Rollback Command**: Available via GitHub Actions
            - **Expected Rollback Time**: < 5 minutes
            
            **⚠️  THREE APPROVALS REQUIRED FOR PRODUCTION DEPLOYMENT**
          exclude-workflow-initiator-as-approver: true
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      # Pre-deployment backup
      - name: Create database backup
        run: |
          echo "Creating production database backup..."
          aws rds create-db-snapshot \
            --db-instance-identifier production-db \
            --db-snapshot-identifier "pre-deployment-$(date +%Y%m%d-%H%M%S)"
          echo "Database backup initiated"
      
      # Canary deployment strategy
      - name: Start canary deployment
        id: canary
        run: |
          echo "Starting canary deployment (5% traffic)..."
          
          # Deploy canary version
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-canary \
            --task-definition devops-app-prod:REVISION \
            --desired-count 1
          
          # Wait for canary to be ready
          aws ecs wait services-stable \
            --cluster production-cluster \
            --services devops-app-canary
          
          # Route 5% traffic to canary
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.MAIN_TARGET_GROUP_ARN }}","Weight":95},
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":5}
              ]
            }'
          
          echo "Canary deployment active with 5% traffic"
          echo "canary_active=true" >> $GITHUB_OUTPUT
      
      # Monitor canary for 10 minutes
      - name: Monitor canary deployment
        run: |
          echo "Monitoring canary deployment for 10 minutes..."
          
          # Monitor error rates and performance
          for i in {1..20}; do
            echo "Canary check $i/20..."
            
            # Check error rate (should be < 1%)
            error_rate=$(aws logs filter-log-events \
              --log-group-name /aws/ecs/production \
              --start-time $(date -d '1 minute ago' +%s)000 \
              --filter-pattern '[timestamp, request_id, "ERROR"]' \
              --query 'length(events)' \
              --output text)
            
            if [ "$error_rate" -gt 10 ]; then
              echo "Error rate too high: $error_rate errors/minute"
              echo "canary_failed=true" >> $GITHUB_OUTPUT
              exit 1
            fi
            
            # Check response time (should be < 500ms average)
            response_time=$(curl -w "%{time_total}" -s -o /dev/null https://example.com/health)
            if (( $(echo "$response_time > 0.5" | bc -l) )); then
              echo "Response time too high: ${response_time}s"
              echo "canary_failed=true" >> $GITHUB_OUTPUT
              exit 1
            fi
            
            echo "Canary metrics OK - Errors: $error_rate, Response Time: ${response_time}s"
            sleep 30
          done
          
          echo "Canary monitoring completed successfully!"
      
      # Promote canary to full deployment
      - name: Promote canary to full deployment
        if: steps.canary.outputs.canary_active == 'true'
        run: |
          echo "Promoting canary to full deployment..."
          
          # Scale up canary to full capacity
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-canary \
            --desired-count 5
          
          # Wait for full scale-up
          aws ecs wait services-stable \
            --cluster production-cluster \
            --services devops-app-canary
          
          # Switch 100% traffic to new version
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":100}
              ]
            }'
          
          # Scale down old version
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-main \
            --desired-count 0
          
          echo "Production deployment completed successfully!"
      
      # Post-deployment verification
      - name: Post-deployment verification
        run: |
          echo "Running post-deployment verification..."
          
          # Comprehensive health checks
          curl -f https://example.com/health
          curl -f https://example.com/api/status
          
          # Critical user journeys
          npm run test:smoke-production
          
          # Performance verification
          npm run test:performance-production
          
          echo "Post-deployment verification completed!"
      
      # Notify successful deployment
      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: success
          channel: '#production-deployments'
          message: |
            🚀 PRODUCTION DEPLOYMENT SUCCESSFUL! 🚀
            
            Environment: Production
            Version: ${{ github.sha }}
            Deployed by: ${{ github.actor }}
            Strategy: Canary (5% → 100%)
            
            🔗 Links:
            • Production: https://example.com
            • Monitoring: https://grafana.example.com
            • Logs: https://kibana.example.com
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # Job 4: Rollback capability
  rollback:
    name: Rollback Production
    runs-on: ubuntu-latest
    environment:
      name: production
    
    # Only run on workflow_dispatch with rollback input
    if: github.event.inputs.rollback == 'true'
    
    steps:
      - name: Emergency rollback approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: platform-leads,on-call-engineer
          minimum-approvals: 1
          issue-title: "🚨 EMERGENCY ROLLBACK APPROVAL"
          issue-body: |
            ## 🚨 EMERGENCY PRODUCTION ROLLBACK 🚨
            
            **Current Version**: ${{ github.sha }}
            **Rollback Target**: Previous stable version
            **Initiated by**: ${{ github.actor }}
            
            **IMMEDIATE APPROVAL REQUIRED**
          exclude-workflow-initiator-as-approver: false
      
      - name: Execute rollback
        run: |
          echo "Executing emergency rollback..."
          
          # Get previous stable version
          previous_version=$(aws ecs describe-services \
            --cluster production-cluster \
            --services devops-app-main \
            --query 'services[0].deployments[1].taskDefinition' \
            --output text)
          
          # Rollback to previous version
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-main \
            --task-definition $previous_version \
            --desired-count 5
          
          # Scale down current version
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-canary \
            --desired-count 0
          
          # Switch traffic back
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.MAIN_TARGET_GROUP_ARN }}","Weight":100}
              ]
            }'
          
          echo "Rollback completed!"
```

### Task 2: GitOps Integration and Configuration Management (90 minutes)

**2.1 GitOps Configuration Updates**

Create workflows that automatically update deployment configurations:

```yaml
# File: .github/workflows/05-gitops-integration.yml
name: GitOps Configuration Management

on:
  push:
    branches: [ main ]
    paths:
      - 'k8s/**'
      - 'helm/**'
      - 'config/**'
  
  workflow_run:
    workflows: ["Comprehensive CI Pipeline"]
    types: [completed]
    branches: [main]

env:
  GITOPS_REPO: devops-team/app-config
  ARGOCD_SERVER: argocd.example.com

jobs:
  # Job 1: Update GitOps repository with new configurations
  update-gitops-config:
    name: Update GitOps Configuration
    runs-on: ubuntu-latest
    if: github.event.workflow_run.conclusion == 'success' || github.event_name == 'push'
    
    steps:
      - name: Checkout source repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops-repo
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.12.0'
      
      # Update development environment configuration
      - name: Update development configuration
        if: contains(github.ref, 'develop')
        run: |
          cd gitops-repo
          
          # Update Helm values for development
          IMAGE_TAG=${{ github.sha }}
          
          # Update frontend image tag
          yq eval '.frontend.image.tag = env(IMAGE_TAG)' -i environments/development/values.yaml
          
          # Update backend image tag
          yq eval '.backend.image.tag = env(IMAGE_TAG)' -i environments/development/values.yaml
          
          # Update configuration version
          yq eval '.global.version = env(IMAGE_TAG)' -i environments/development/values.yaml
          
          echo "Development configuration updated with image tag: $IMAGE_TAG"
      
      # Update staging environment configuration
      - name: Update staging configuration
        if: github.ref == 'refs/heads/main'
        run: |
          cd gitops-repo
          
          IMAGE_TAG=${{ github.sha }}
          
          # Update staging environment
          yq eval '.frontend.image.tag = env(IMAGE_TAG)' -i environments/staging/values.yaml
          yq eval '.backend.image.tag = env(IMAGE_TAG)' -i environments/staging/values.yaml
          yq eval '.global.version = env(IMAGE_TAG)' -i environments/staging/values.yaml
          
          # Update resource limits for staging
          yq eval '.frontend.resources.limits.memory = "512Mi"' -i environments/staging/values.yaml
          yq eval '.backend.resources.limits.memory = "1Gi"' -i environments/staging/values.yaml
          
          echo "Staging configuration updated with image tag: $IMAGE_TAG"
      
      # Validate Helm charts
      - name: Validate Helm charts
        run: |
          cd gitops-repo
          
          # Lint Helm charts
          helm lint charts/devops-app/
          
          # Template and validate for each environment
          for env in development staging production; do
            echo "Validating $env environment..."
            helm template devops-app charts/devops-app/ \
              -f environments/$env/values.yaml \
              --output-dir /tmp/$env
            
            # Validate Kubernetes manifests
            kubectl --dry-run=client apply -R -f /tmp/$env/
          done
          
          echo "All Helm charts validated successfully"
      
      # Commit changes to GitOps repository
      - name: Commit and push changes
        run: |
          cd gitops-repo
          
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          
          git add .
          
          if git diff --staged --quiet; then
            echo "No changes to commit"
          else
            git commit -m "chore: update configuration for ${{ github.sha }}
            
            - Updated image tags for all services
            - Source commit: ${{ github.sha }}
            - Triggered by: ${{ github.actor }}
            - Repository: ${{ github.repository }}"
            
            git push
            echo "GitOps configuration updated and pushed"
          fi
      
      # Trigger ArgoCD sync (optional)
      - name: Sync ArgoCD applications
        run: |
          # Install ArgoCD CLI
          curl -sSL -o /tmp/argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x /tmp/argocd-linux-amd64
          sudo mv /tmp/argocd-linux-amd64 /usr/local/bin/argocd
          
          # Login to ArgoCD
          argocd login ${{ env.ARGOCD_SERVER }} \
            --username ${{ secrets.ARGOCD_USERNAME }} \
            --password ${{ secrets.ARGOCD_PASSWORD }} \
            --insecure
          
          # Sync applications
          argocd app sync devops-app-development
          argocd app sync devops-app-staging
          
          # Wait for sync to complete
          argocd app wait devops-app-development --health
          argocd app wait devops-app-staging --health
          
          echo "ArgoCD applications synced successfully"

  # Job 2: Multi-environment promotion pipeline
  promote-environments:
    name: Environment Promotion
    runs-on: ubuntu-latest
    needs: [update-gitops-config]
    
    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops-repo
      
      # Promote from staging to production (manual approval required)
      - name: Wait for production promotion approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: platform-leads,release-managers
          minimum-approvals: 2
          issue-title: "Production Promotion Approval"
          issue-body: |
            ## Production Promotion Request
            
            **Version**: ${{ github.sha }}
            **Environment**: staging → production
            
            ### Validation Checklist:
            - [ ] Staging deployment successful
            - [ ] All integration tests passed
            - [ ] Performance benchmarks met
            - [ ] Security scans completed
            - [ ] Database migrations tested
            
            **Approve to promote to production**
      
      - name: Promote to production
        run: |
          cd gitops-repo
          
          # Copy staging configuration to production
          cp environments/staging/values.yaml environments/production/values.yaml
          
          # Update production-specific settings
          yq eval '.global.environment = "production"' -i environments/production/values.yaml
          yq eval '.ingress.host = "example.com"' -i environments/production/values.yaml
          yq eval '.replicas.frontend = 5' -i environments/production/values.yaml
          yq eval '.replicas.backend = 5' -i environments/production/values.yaml
          
          # Update resource limits for production
          yq eval '.frontend.resources.limits.memory = "1Gi"' -i environments/production/values.yaml
          yq eval '.backend.resources.limits.memory = "2Gi"' -i environments/production/values.yaml
          
          # Commit changes
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          
          git add environments/production/values.yaml
          git commit -m "feat: promote ${{ github.sha }} to production
          
          - Promoted from staging to production
          - Updated production configuration
          - Ready for deployment"
          
          git push
          
          echo "Configuration promoted to production"

  # Job 3: Configuration validation and drift detection
  validate-config:
    name: Configuration Validation
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
      
      # Validate configuration consistency
      - name: Validate configuration consistency
        run: |
          echo "Validating configuration consistency across environments..."
          
          # Check that all environments have required fields
          for env in development staging production; do
            echo "Validating $env environment..."
            
            # Check required fields
            yq eval '.frontend.image.repository' environments/$env/values.yaml
            yq eval '.backend.image.repository' environments/$env/values.yaml
            yq eval '.global.version' environments/$env/values.yaml
            
            # Validate resource limits are set
            if [ "$env" != "development" ]; then
              yq eval '.frontend.resources.limits.memory' environments/$env/values.yaml
              yq eval '.backend.resources.limits.memory' environments/$env/values.yaml
            fi
          done
          
          echo "Configuration validation completed"
      
      # Check for configuration drift
      - name: Detect configuration drift
        run: |
          echo "Checking for configuration drift..."
          
          # Compare live cluster state with Git configuration
          kubectl get deployments -n devops-app-production -o yaml > /tmp/live-config.yaml
          helm template devops-app charts/devops-app/ \
            -f environments/production/values.yaml \
            --namespace devops-app-production > /tmp/desired-config.yaml
          
          # Compare configurations (simplified check)
          if diff -q /tmp/live-config.yaml /tmp/desired-config.yaml > /dev/null; then
            echo "No configuration drift detected"
          else
            echo "⚠️  Configuration drift detected!"
            echo "Live configuration differs from Git configuration"
            
            # Create issue for drift detection
            gh issue create \
              --title "Configuration Drift Detected" \
              --body "Configuration drift detected in production environment. Please review and sync." \
              --assignee platform-team
          fi
```

### Task 3: Advanced Deployment Strategies and Monitoring (90 minutes)

**2.3 Feature Flag Integration and Progressive Deployment**

```yaml
# File: .github/workflows/06-progressive-deployment.yml
name: Progressive Deployment with Feature Flags

on:
  workflow_dispatch:
    inputs:
      feature_flag:
        description: 'Feature flag to enable'
        required: false
        type: string
      deployment_strategy:
        description: 'Deployment strategy'
        required: true
        type: choice
        default: 'canary'
        options:
        - canary
        - blue-green
        - rolling
        - feature-flag

env:
  LAUNCHDARKLY_PROJECT_KEY: devops-app
  DATADOG_SITE: datadoghq.com

jobs:
  # Job 1: Feature flag deployment
  feature-flag-deployment:
    name: Feature Flag Deployment
    runs-on: ubuntu-latest
    if: github.event.inputs.deployment_strategy == 'feature-flag'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Update feature flag configuration
      - name: Update feature flags
        run: |
          # Use LaunchDarkly API to update feature flags
          curl -X PATCH "https://app.launchdarkly.com/api/v2/flags/${{ env.LAUNCHDARKLY_PROJECT_KEY }}/${{ github.event.inputs.feature_flag }}" \
            -H "Authorization: ${{ secrets.LAUNCHDARKLY_ACCESS_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "patch": [
                {
                  "op": "replace",
                  "path": "/environments/production/on",
                  "value": true
                },
                {
                  "op": "replace", 
                  "path": "/environments/production/targets/0/values",
                  "value": [true]
                }
              ]
            }'
          
          echo "Feature flag ${{ github.event.inputs.feature_flag }} enabled in production"
      
      # Monitor feature flag rollout
      - name: Monitor feature flag rollout
        run: |
          echo "Monitoring feature flag rollout for 30 minutes..."
          
          for i in {1..60}; do
            # Check error rates via Datadog API
            error_rate=$(curl -X GET \
              "https://api.${{ env.DATADOG_SITE }}/api/v1/query" \
              -H "Content-Type: application/json" \
              -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
              -H "DD-APPLICATION-KEY: ${{ secrets.DATADOG_APP_KEY }}" \
              -G --data-urlencode "query=avg:trace.http.request.errors{env:production,feature:${{ github.event.inputs.feature_flag }}}" \
              | jq '.series[0].pointlist[-1][1]')
            
            if (( $(echo "$error_rate > 0.05" | bc -l) )); then
              echo "Error rate too high: $error_rate"
              
              # Disable feature flag
              curl -X PATCH "https://app.launchdarkly.com/api/v2/flags/${{ env.LAUNCHDARKLY_PROJECT_KEY }}/${{ github.event.inputs.feature_flag }}" \
                -H "Authorization: ${{ secrets.LAUNCHDARKLY_ACCESS_TOKEN }}" \
                -H "Content-Type: application/json" \
                -d '{"patch": [{"op": "replace", "path": "/environments/production/on", "value": false}]}'
              
              echo "Feature flag automatically disabled due to high error rate"
              exit 1
            fi
            
            echo "Monitor check $i/60 - Error rate: $error_rate"
            sleep 30
          done
          
          echo "Feature flag rollout monitoring completed successfully"

  # Job 2: Progressive canary deployment with automatic promotion
  progressive-canary:
    name: Progressive Canary Deployment
    runs-on: ubuntu-latest
    if: github.event.inputs.deployment_strategy == 'canary'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      # Stage 1: 5% canary
      - name: Deploy 5% canary
        run: |
          echo "Deploying 5% canary..."
          
          # Update canary service
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-canary \
            --task-definition devops-app-prod:REVISION
          
          # Route 5% traffic to canary
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.MAIN_TARGET_GROUP_ARN }}","Weight":95},
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":5}
              ]
            }'
      
      - name: Monitor 5% canary for 10 minutes
        run: |
          ./scripts/monitor-deployment.sh 5 10
      
      # Stage 2: 25% canary
      - name: Promote to 25% canary
        run: |
          echo "Promoting to 25% canary..."
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.MAIN_TARGET_GROUP_ARN }}","Weight":75},
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":25}
              ]
            }'
      
      - name: Monitor 25% canary for 15 minutes
        run: |
          ./scripts/monitor-deployment.sh 25 15
      
      # Stage 3: 50% canary
      - name: Promote to 50% canary
        run: |
          echo "Promoting to 50% canary..."
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.MAIN_TARGET_GROUP_ARN }}","Weight":50},
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":50}
              ]
            }'
      
      - name: Monitor 50% canary for 20 minutes
        run: |
          ./scripts/monitor-deployment.sh 50 20
      
      # Stage 4: Full promotion
      - name: Promote to 100% (complete deployment)
        run: |
          echo "Completing canary deployment (100% traffic)..."
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.CANARY_RULE_ARN }} \
            --actions Type=forward,ForwardConfig='{
              "TargetGroups":[
                {"TargetGroupArn":"${{ secrets.CANARY_TARGET_GROUP_ARN }}","Weight":100}
              ]
            }'
          
          # Scale down old version
          aws ecs update-service \
            --cluster production-cluster \
            --service devops-app-main \
            --desired-count 0
          
          echo "Progressive canary deployment completed successfully!"
      
      # Final verification
      - name: Post-deployment verification
        run: |
          ./scripts/comprehensive-verification.sh
          
          # Update deployment status in monitoring
          curl -X POST "https://api.${{ env.DATADOG_SITE }}/api/v1/events" \
            -H "Content-Type: application/json" \
            -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
            -d '{
              "title": "Production Deployment Completed",
              "text": "Progressive canary deployment completed successfully",
              "tags": ["environment:production", "deployment:canary", "version:${{ github.sha }}"],
              "alert_type": "success"
            }'

  # Job 3: Automated rollback based on SLI/SLO violations
  automated-rollback:
    name: Automated Rollback Monitor
    runs-on: ubuntu-latest
    needs: [progressive-canary]
    if: always()
    
    steps:
      - name: Monitor SLI/SLO compliance
        run: |
          echo "Monitoring SLI/SLO compliance for 1 hour post-deployment..."
          
          for i in {1..12}; do  # Monitor for 1 hour (12 * 5 minutes)
            # Check availability SLO (99.9% uptime)
            availability=$(curl -s "https://api.${{ env.DATADOG_SITE }}/api/v1/query" \
              -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
              -H "DD-APPLICATION-KEY: ${{ secrets.DATADOG_APP_KEY }}" \
              -G --data-urlencode "query=avg:trace.http.request.hits{env:production,status:ok}/avg:trace.http.request.hits{env:production}" \
              | jq '.series[0].pointlist[-1][1]')
            
            # Check latency SLO (95% of requests < 500ms)
            p95_latency=$(curl -s "https://api.${{ env.DATADOG_SITE }}/api/v1/query" \
              -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
              -H "DD-APPLICATION-KEY: ${{ secrets.DATADOG_APP_KEY }}" \
              -G --data-urlencode "query=p95:trace.http.request.duration{env:production}" \
              | jq '.series[0].pointlist[-1][1]')
            
            # Check error rate SLO (< 0.1% error rate)
            error_rate=$(curl -s "https://api.${{ env.DATADOG_SITE }}/api/v1/query" \
              -H "DD-API-KEY: ${{ secrets.DATADOG_API_KEY }}" \
              -H "DD-APPLICATION-KEY: ${{ secrets.DATADOG_APP_KEY }}" \
              -G --data-urlencode "query=avg:trace.http.request.errors{env:production}/avg:trace.http.request.hits{env:production}" \
              | jq '.series[0].pointlist[-1][1]')
            
            echo "SLI Check $i/12:"
            echo "  Availability: $availability (SLO: > 0.999)"
            echo "  P95 Latency: $p95_latency ms (SLO: < 500)"  
            echo "  Error Rate: $error_rate (SLO: < 0.001)"
            
            # Check if any SLO is violated
            if (( $(echo "$availability < 0.999" | bc -l) )) || \
               (( $(echo "$p95_latency > 500" | bc -l) )) || \
               (( $(echo "$error_rate > 0.001" | bc -l) )); then
              
              echo "🚨 SLO VIOLATION DETECTED! Initiating automatic rollback..."
              
              # Trigger rollback workflow
              curl -X POST \
                -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
                -H "Accept: application/vnd.github.v3+json" \
                "https://api.github.com/repos/${{ github.repository }}/actions/workflows/04-environment-management.yml/dispatches" \
                -d '{"ref":"main","inputs":{"rollback":"true","reason":"SLO violation detected"}}'
              
              exit 1
            fi
            
            sleep 300  # Wait 5 minutes before next check
          done
          
          echo "✅ SLO compliance monitoring completed - All SLOs met!"
```

## Deliverables

1. **Advanced Deployment Workflows:**
   - Environment-specific deployment with approval gates
   - Blue-green and canary deployment strategies
   - Automated rollback based on health metrics
   - Feature flag integration and progressive rollouts

2. **GitOps Integration:**
   - Automated configuration management
   - Multi-environment promotion pipelines
   - Configuration drift detection and remediation
   - ArgoCD synchronization and monitoring

3. **Production Monitoring Integration:**
   - SLI/SLO monitoring and alerting
   - Automated rollback on metric violations
   - Comprehensive deployment health tracking
   - Integration with observability platforms

4. **Enterprise-Grade Security:**
   - Multi-level approval workflows
   - Secure secret management
   - Audit trails and compliance logging
   - Automated security scanning integration

## Success Criteria

- Successfully implemented automated deployment to multiple environments
- Created approval workflows that enforce proper change management
- Integrated monitoring and automated rollback capabilities
- Demonstrated GitOps practices with configuration management
- Built production-ready deployment pipelines with proper safety measures
- Implemented advanced deployment strategies (canary, blue-green)
- Created comprehensive rollback and incident response procedures

## Verification Commands

```bash
# Test deployment workflows
gh workflow run "Environment Management & Deployment" \
  --field environment=staging \
  --field force_deploy=false

# Monitor deployment status
gh run watch

# Test GitOps integration
git commit -m "test: trigger GitOps workflow" --allow-empty
git push

# Check ArgoCD applications
argocd app list
argocd app get devops-app-production

# Test rollback procedure
gh workflow run "Environment Management & Deployment" \
  --field rollback=true
```

## Integration with DevOps Pipeline

This advanced deployment automation provides:
- **Production Readiness**: Enterprise-grade deployment practices
- **Risk Mitigation**: Automated testing and rollback capabilities
- **Compliance**: Approval workflows and audit trails
- **Observability**: Comprehensive monitoring and alerting
- **Scalability**: Multi-environment and multi-service support

This exercise provides 240 minutes of intensive, production-focused deployment automation experience, completing the comprehensive GitHub Actions curriculum!