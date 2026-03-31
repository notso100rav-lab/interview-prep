# CI/CD Pipelines

> Java Backend Engineer Interview Prep — Chapter 4.4

---

## Table of Contents
1. [GitHub Actions — Full Workflow](#github-actions--full-workflow)
2. [Jenkins Declarative Pipeline](#jenkins-declarative-pipeline)
3. [Blue-Green Deployment](#blue-green-deployment)
4. [Canary Deployment](#canary-deployment)
5. [Q&A](#qa)

---

## GitHub Actions — Full Workflow

### Complete Spring Boot CI/CD Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]
  workflow_dispatch:       # allow manual trigger

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: myapp-cluster
  ECS_SERVICE: myapp-service
  CONTAINER_NAME: myapp
  JAVA_VERSION: '17'

jobs:

  # ────────────────────────────────────────────────────────────────
  # JOB 1: Test
  # ────────────────────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      checks: write      # for test report publishing

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven           # cache Maven dependencies

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Run tests
        run: mvn --no-transfer-progress test
        env:
          SPRING_PROFILES_ACTIVE: test

      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Maven Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit

      - name: Generate code coverage report
        run: mvn jacoco:report

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: target/site/jacoco/jacoco.xml
          token: ${{ secrets.CODECOV_TOKEN }}

  # ────────────────────────────────────────────────────────────────
  # JOB 2: Build & Push Docker Image to ECR
  # ────────────────────────────────────────────────────────────────
  build-and-push:
    name: Build & Push to ECR
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write    # for OIDC AWS auth

    outputs:
      image-uri: ${{ steps.build-push.outputs.image-uri }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build JAR (skip tests — already tested)
        run: mvn --no-transfer-progress package -DskipTests

      - name: Configure AWS credentials (OIDC — no stored keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            APP_VERSION=${{ github.sha }}
            BUILD_DATE=${{ github.run_id }}

      - name: Output image URI
        run: |
          echo "image-uri=${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}:sha-${{ github.sha }}" >> $GITHUB_OUTPUT

  # ────────────────────────────────────────────────────────────────
  # JOB 3: Deploy to ECS
  # ────────────────────────────────────────────────────────────────
  deploy-ecs:
    name: Deploy to ECS
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.myapp.com
    permissions:
      id-token: write

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_SERVICE }} \
            --query taskDefinition > task-definition.json

      - name: Fill in the new image ID in the task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build-and-push.outputs.image-uri }}

      - name: Deploy to ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 10

  # ────────────────────────────────────────────────────────────────
  # JOB 4: Deploy to Kubernetes (alternative to ECS job)
  # ────────────────────────────────────────────────────────────────
  deploy-k8s:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-push
    if: false   # toggle this on if using K8s instead of ECS
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name myapp-cluster \
            --region ${{ env.AWS_REGION }}

      - name: Update image in deployment
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build-and-push.outputs.image-uri }} \
            -n production

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/myapp \
            -n production \
            --timeout=5m

      - name: Verify deployment
        run: |
          kubectl get pods -n production -l app=myapp
```

### GitHub Actions Secrets Setup

```bash
# Set secrets via GitHub CLI
gh secret set AWS_ACCOUNT_ID --body "123456789012"
gh secret set CODECOV_TOKEN --body "your-codecov-token"
gh secret set SLACK_WEBHOOK_URL --body "https://hooks.slack.com/..."

# List secrets
gh secret list
```

### OIDC Trust Policy for GitHub Actions (No Static Keys)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myapp:*"
        }
      }
    }
  ]
}
```

---

## Jenkins Declarative Pipeline

### Full Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }

    environment {
        APP_NAME = 'myapp'
        ECR_REGISTRY = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPO = "${ECR_REGISTRY}/${APP_NAME}"
        AWS_REGION = 'us-east-1'
        ECS_CLUSTER = 'myapp-cluster'
        ECS_SERVICE = 'myapp-service'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        FULL_IMAGE = "${ECR_REPO}:${IMAGE_TAG}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                script {
                    def gitInfo = sh(
                        script: "git log --oneline -1",
                        returnStdout: true
                    ).trim()
                    echo "Building commit: ${gitInfo}"
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn --no-transfer-progress test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco(
                        execPattern: 'target/*.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java'
                    )
                }
            }
        }

        stage('Code Analysis') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh 'mvn sonar:sonar'
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }
                stage('Dependency Check') {
                    steps {
                        sh 'mvn org.owasp:dependency-check-maven:check'
                    }
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn --no-transfer-progress package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    docker.build("${FULL_IMAGE}", "--build-arg APP_VERSION=${IMAGE_TAG} .")
                    docker.build("${ECR_REPO}:latest", ".")
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${FULL_IMAGE}"
                }
            }
        }

        stage('Push to ECR') {
            when {
                branch 'main'
            }
            steps {
                withAWS(region: env.AWS_REGION, credentials: 'aws-jenkins-role') {
                    script {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                        sh "docker push ${FULL_IMAGE}"
                        sh "docker push ${ECR_REPO}:latest"
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            environment {
                ECS_SERVICE = 'myapp-staging-service'
            }
            steps {
                withAWS(region: env.AWS_REGION, credentials: 'aws-jenkins-role') {
                    script {
                        updateECSService(env.ECS_CLUSTER, env.ECS_SERVICE, env.FULL_IMAGE)
                    }
                }
            }
        }

        stage('Integration Tests') {
            when {
                branch 'main'
            }
            steps {
                sh 'mvn --no-transfer-progress verify -Pintegration-tests -Dapp.url=https://staging.myapp.com'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
                submitter "devops-team,senior-engineers"
            }
            environment {
                ECS_SERVICE = 'myapp-prod-service'
            }
            steps {
                withAWS(region: env.AWS_REGION, credentials: 'aws-jenkins-role') {
                    script {
                        updateECSService(env.ECS_CLUSTER, env.ECS_SERVICE, env.FULL_IMAGE)
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ *${APP_NAME}* deployed successfully. Version: `${IMAGE_TAG}`"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ *${APP_NAME}* pipeline failed. Build: `${env.BUILD_URL}`"
            )
        }
        always {
            cleanWs()
        }
    }
}

// Helper function
def updateECSService(String cluster, String service, String image) {
    sh """
        TASK_DEF=\$(aws ecs describe-task-definition \
            --task-definition ${service} \
            --query taskDefinition)

        NEW_TASK_DEF=\$(echo \$TASK_DEF | jq \
            --arg IMAGE "${image}" \
            '.containerDefinitions[0].image = \$IMAGE | del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)')

        NEW_TASK_ARN=\$(aws ecs register-task-definition \
            --cli-input-json "\$NEW_TASK_DEF" \
            --query taskDefinition.taskDefinitionArn \
            --output text)

        aws ecs update-service \
            --cluster ${cluster} \
            --service ${service} \
            --task-definition \$NEW_TASK_ARN \
            --force-new-deployment

        aws ecs wait services-stable \
            --cluster ${cluster} \
            --services ${service}
    """
}
```

---

## Blue-Green Deployment

### Architecture

```
                    ┌─────────────────────────────────┐
                    │           ALB Listener           │
                    └────────────────┬────────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │         Listener Rule            │
                    │   Forward to: [BLUE | GREEN]     │
                    └──────────┬──────────────┬───────┘
                               │              │
              ┌────────────────▼──┐    ┌──────▼────────────┐
              │  TG: BLUE (v1)    │    │  TG: GREEN (v2)   │
              │  (current/live)   │    │  (new/staging)    │
              └────────────────┬──┘    └──────┬────────────┘
                               │              │
              ┌────────────────▼──┐    ┌──────▼────────────┐
              │  ECS Tasks v1.0   │    │  ECS Tasks v2.0   │
              └───────────────────┘    └───────────────────┘
```

### Blue-Green GitHub Actions Workflow

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deploy

on:
  workflow_dispatch:
    inputs:
      image-tag:
        description: 'Docker image tag to deploy'
        required: true

jobs:
  blue-green-deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE }}
          aws-region: us-east-1

      - name: Determine current active color
        id: get-color
        run: |
          ACTIVE_TG=$(aws elbv2 describe-listeners \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --query 'Listeners[0].DefaultActions[0].TargetGroupArn' \
            --output text)

          if echo "$ACTIVE_TG" | grep -q "blue"; then
            echo "active=blue" >> $GITHUB_OUTPUT
            echo "inactive=green" >> $GITHUB_OUTPUT
            echo "inactive-tg-arn=${{ secrets.GREEN_TG_ARN }}" >> $GITHUB_OUTPUT
          else
            echo "active=green" >> $GITHUB_OUTPUT
            echo "inactive=blue" >> $GITHUB_OUTPUT
            echo "inactive-tg-arn=${{ secrets.BLUE_TG_ARN }}" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to inactive slot
        run: |
          # Update inactive ECS service with new image
          aws ecs update-service \
            --cluster myapp-cluster \
            --service myapp-${{ steps.get-color.outputs.inactive }} \
            --task-definition $(aws ecs describe-task-definition \
              --task-definition myapp \
              --query "taskDefinition.taskDefinitionArn" \
              --output text) \
            --force-new-deployment

          # Wait for stability
          aws ecs wait services-stable \
            --cluster myapp-cluster \
            --services myapp-${{ steps.get-color.outputs.inactive }}

      - name: Health check on inactive slot
        run: |
          TARGET_GROUP_ARN=${{ steps.get-color.outputs.inactive-tg-arn }}

          # Wait for all targets to be healthy
          for i in $(seq 1 30); do
            UNHEALTHY=$(aws elbv2 describe-target-health \
              --target-group-arn $TARGET_GROUP_ARN \
              --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]' \
              --output text)

            if [ -z "$UNHEALTHY" ]; then
              echo "All targets healthy!"
              exit 0
            fi
            echo "Waiting for targets... attempt $i/30"
            sleep 10
          done
          echo "Health check timeout!" && exit 1

      - name: Switch traffic to new slot
        run: |
          aws elbv2 modify-listener \
            --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
            --default-actions Type=forward,TargetGroupArn=${{ steps.get-color.outputs.inactive-tg-arn }}

          echo "Traffic switched to ${{ steps.get-color.outputs.inactive }} slot"

      - name: Smoke test production
        run: |
          sleep 15  # let traffic settle
          curl -f https://api.myapp.com/actuator/health || (echo "Smoke test failed!" && exit 1)

      - name: Scale down old slot
        run: |
          aws ecs update-service \
            --cluster myapp-cluster \
            --service myapp-${{ steps.get-color.outputs.active }} \
            --desired-count 0
```

### Rollback in Blue-Green

```bash
# Instant rollback — just flip the listener back
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --default-actions Type=forward,TargetGroupArn=arn:...:targetgroup/myapp-blue/...

echo "Rolled back to blue in seconds"
```

---

## Canary Deployment

### Weighted Target Groups (AWS ALB)

```yaml
# .github/workflows/canary-deploy.yml
- name: Start canary (10% traffic to new version)
  run: |
    aws elbv2 modify-listener \
      --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
      --default-actions '[
        {
          "Type": "forward",
          "ForwardConfig": {
            "TargetGroups": [
              {"TargetGroupArn": "${{ secrets.STABLE_TG_ARN }}", "Weight": 90},
              {"TargetGroupArn": "${{ secrets.CANARY_TG_ARN }}", "Weight": 10}
            ]
          }
        }
      ]'

- name: Monitor error rate for 5 minutes
  run: |
    for i in $(seq 1 5); do
      ERROR_RATE=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/ApplicationELB \
        --metric-name HTTPCode_Target_5XX_Count \
        --dimensions Name=TargetGroup,Value=${{ secrets.CANARY_TG_ARN }} \
        --start-time $(date -u -d '1 minute ago' '+%Y-%m-%dT%H:%M:%SZ') \
        --end-time $(date -u '+%Y-%m-%dT%H:%M:%SZ') \
        --period 60 --statistics Sum \
        --query 'Datapoints[0].Sum' --output text)
      echo "5xx errors in last minute: $ERROR_RATE"
      sleep 60
    done

- name: Promote canary to 100% (or rollback)
  run: |
    if [ "$ERROR_RATE" -gt "10" ]; then
      # Rollback
      aws elbv2 modify-listener \
        --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
        --default-actions Type=forward,TargetGroupArn=${{ secrets.STABLE_TG_ARN }}
    else
      # Promote
      aws elbv2 modify-listener \
        --listener-arn ${{ secrets.ALB_LISTENER_ARN }} \
        --default-actions Type=forward,TargetGroupArn=${{ secrets.CANARY_TG_ARN }}
    fi
```

---

## Q&A

### Q1 🟢 What is the difference between CI and CD?

<details><summary>Click to reveal answer</summary>

| Term | Full Name | Goal |
|------|-----------|------|
| **CI** | Continuous Integration | Automatically build and test code on every commit |
| **CD** | Continuous Delivery | Automatically prepare release artifact (manual deploy trigger) |
| **CD** | Continuous Deployment | Automatically deploy to production without human intervention |

**CI pipeline**: commit → build → unit test → code analysis → artifact

**CD pipeline**: artifact → deploy to staging → integration tests → [manual approval] → deploy to production

GitHub Actions enables both. The `environment: production` with required reviewers in GitHub provides the manual approval gate for Continuous Delivery.

</details>

---

### Q2 🟢 How does GitHub Actions cache Maven dependencies?

<details><summary>Click to reveal answer</summary>

```yaml
- name: Set up JDK with Maven cache
  uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: temurin
    cache: maven    # built-in caching shortcut

# OR explicit cache control:
- name: Cache Maven repository
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-
```

**How it works:**
- Cache key includes hash of all `pom.xml` files
- If any pom.xml changes, the cache key changes → full download
- `restore-keys` provides fallback to any previous Maven cache
- Cache is shared across workflow runs in the same repo

Reduces build time from ~3 min to ~30s for dependency resolution.

</details>

---

### Q3 🟡 Explain the difference between `push`, `pull_request`, and `workflow_dispatch` triggers.

<details><summary>Click to reveal answer</summary>

```yaml
on:
  push:
    branches: [main, develop]
    tags: ['v*.*.*']
    paths:
      - 'src/**'       # only trigger if source files changed
      - 'pom.xml'

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
    paths-ignore:
      - 'docs/**'      # don't trigger for docs-only PRs

  schedule:
    - cron: '0 2 * * *'   # nightly at 2 AM UTC

  workflow_dispatch:      # manual trigger via UI or API
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging
      dry-run:
        type: boolean
        default: false
```

| Trigger | Use Case |
|---------|----------|
| `push` | Full CI/CD on merge to main |
| `pull_request` | Validation on PRs (test, lint, no deploy) |
| `schedule` | Nightly security scans, integration tests |
| `workflow_dispatch` | Manual deployments, hotfixes |

</details>

---

### Q4 🟡 How do you use GitHub Actions environments and why?

<details><summary>Click to reveal answer</summary>

Environments provide:
1. **Protection rules** — required reviewers before deploy
2. **Environment-specific secrets** — prod DB password only in prod environment
3. **Deployment URL** — shown in PR status

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://api.myapp.com   # shown in GitHub UI
```

```yaml
# Configure in GitHub Settings → Environments → production:
# - Required reviewers: @devops-lead @senior-engineer
# - Wait timer: 5 minutes
# - Deployment branches: only main
# - Secrets: AWS_PROD_ROLE_ARN, DB_PASSWORD
```

```yaml
# Environment secrets override repo secrets
- name: Configure AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    # uses environment secret "AWS_ROLE_ARN" if in an environment,
    # otherwise falls back to repo-level secret
```

</details>

---

### Q5 🔴 How do you implement matrix builds in GitHub Actions?

<details><summary>Click to reveal answer</summary>

Matrix strategy runs jobs in parallel across multiple configurations:

```yaml
jobs:
  test:
    strategy:
      fail-fast: false   # don't cancel other matrix jobs if one fails
      matrix:
        java: [11, 17, 21]
        os: [ubuntu-latest, windows-latest]
        exclude:
          - java: 11
            os: windows-latest   # skip this combination
        include:
          - java: 17
            os: ubuntu-latest
            coverage: true       # extra variable for specific combination

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}

      - run: mvn test

      - name: Upload coverage (only for java 17 on ubuntu)
        if: matrix.coverage
        run: mvn jacoco:report
```

Useful for: testing against multiple Java versions, multiple databases, multiple OS.

</details>

---

### Q6 🟡 What is the difference between blue-green and canary deployments?

<details><summary>Click to reveal answer</summary>

| Feature | Blue-Green | Canary |
|---------|-----------|--------|
| **Traffic switch** | Instant (0% → 100%) | Gradual (1% → 10% → 50% → 100%) |
| **Risk** | Higher (all traffic hits new) | Lower (small % initially) |
| **Cost** | 2x resources | Slightly more than 1x |
| **Rollback** | Instant (flip LB) | Quick (route back to stable) |
| **Testing** | Pre-switch health check | Real production traffic analysis |
| **Use case** | Major releases, schema changes | High-risk changes, new features |

**Blue-Green**: Two identical environments. Switch all traffic at once. Keep old running for instant rollback.

**Canary**: Route small % of real users to new version. Monitor error rate, latency. Gradually increase if healthy.

Both require: load balancer with weighted routing, health checks, monitoring.

</details>

---

### Q7 🔴 How do you handle database migrations in a CI/CD pipeline?

<details><summary>Click to reveal answer</summary>

**Rule**: Migrations must be **backward compatible** during rolling deploys — the old version must work with the new schema.

**Safe migration patterns:**

1. **Expand-Contract pattern:**
   - Release 1: Add new nullable column (old code ignores it)
   - Release 2: Deploy new code using new column
   - Release 3: Drop old column / add NOT NULL constraint

2. **Flyway in CI/CD:**
```yaml
# GitHub Actions
- name: Run DB migrations
  run: |
    mvn flyway:migrate \
      -Dflyway.url=${{ secrets.DB_URL }} \
      -Dflyway.user=${{ secrets.DB_USER }} \
      -Dflyway.password=${{ secrets.DB_PASS }}

- name: Deploy app
  run: kubectl apply -f k8s/deployment.yaml
```

3. **Separate migration job (Kubernetes):**
```yaml
# k8s/job-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: flyway
          image: flyway/flyway:latest
          command: ["flyway", "migrate"]
          env:
            - name: FLYWAY_URL
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: DB_URL
```

</details>

---

### Q8 🟡 How do you prevent secrets from being logged in GitHub Actions?

<details><summary>Click to reveal answer</summary>

GitHub Actions automatically masks secrets referenced via `${{ secrets.NAME }}` in logs.

```yaml
steps:
  - name: Login to ECR
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # *** in logs
    run: |
      echo "Password: $DB_PASSWORD"   # prints "Password: ***"
```

**Best practices:**

```yaml
# ✅ Use secrets context — automatically masked
env:
  TOKEN: ${{ secrets.API_TOKEN }}

# ✅ Mask dynamic values with ::add-mask::
- name: Get dynamic token
  run: |
    TOKEN=$(aws secretsmanager get-secret-value --secret-id mytoken --query SecretString --output text)
    echo "::add-mask::$TOKEN"    # mask before any subsequent output
    echo "TOKEN=$TOKEN" >> $GITHUB_ENV

# ❌ Never echo secrets directly
- run: echo ${{ secrets.DB_PASSWORD }}   # still masked, but bad practice

# ❌ Never store secrets in artifacts
- uses: actions/upload-artifact@v4
  with:
    path: config.yml   # if config.yml contains secrets, they're exposed!
```

</details>

---

### Q9 🔴 How do you implement a pipeline that only deploys if all quality gates pass?

<details><summary>Click to reveal answer</summary>

```yaml
jobs:
  quality-gate:
    runs-on: ubuntu-latest
    outputs:
      passed: ${{ steps.gate.outputs.passed }}

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: temurin

      - name: Run tests with coverage
        run: mvn verify jacoco:report

      - name: Check coverage threshold
        id: coverage-check
        run: |
          COVERAGE=$(python3 -c "
          import xml.etree.ElementTree as ET
          tree = ET.parse('target/site/jacoco/jacoco.xml')
          root = tree.getroot()
          counters = {c.get('type'): c for c in root.findall('counter')}
          covered = int(counters['LINE'].get('covered', 0))
          missed = int(counters['LINE'].get('missed', 0))
          total = covered + missed
          print(f'{covered/total*100:.1f}' if total > 0 else '0')
          ")
          echo "coverage=$COVERAGE" >> $GITHUB_OUTPUT
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 80%"
            exit 1
          fi

      - name: SonarQube gate
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=${{ secrets.SONAR_HOST }} \
            -Dsonar.token=${{ secrets.SONAR_TOKEN }}
          # Check quality gate status via API
          sleep 30
          STATUS=$(curl -su "${{ secrets.SONAR_TOKEN }}:" \
            "${{ secrets.SONAR_HOST }}/api/qualitygates/project_status?projectKey=myapp" \
            | jq -r '.projectStatus.status')
          [ "$STATUS" = "OK" ] || (echo "SonarQube gate failed: $STATUS" && exit 1)

      - name: Security scan
        run: |
          mvn org.owasp:dependency-check-maven:check \
            -DfailBuildOnCVSS=7

  deploy:
    needs: quality-gate
    if: needs.quality-gate.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "All quality gates passed, deploying..."
```

</details>

---

### Q10 🟡 What is the purpose of `actions/cache` and how does it work?

<details><summary>Click to reveal answer</summary>

`actions/cache` saves and restores directories between workflow runs to speed up builds.

```yaml
- name: Cache Maven deps
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository         # what to cache
    key: maven-${{ hashFiles('**/pom.xml') }}  # cache key
    restore-keys: |                # fallback keys (partial match)
      maven-

- name: Cache Gradle
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: gradle-
```

**How it works:**
1. **Restore**: At start of step, looks for cache matching `key`. Falls back to `restore-keys` (prefix match).
2. **Save**: After job completes, saves `path` under `key`. Won't overwrite existing cache with same key.

**Cache invalidation**: Change `key` (e.g., by updating pom.xml → hash changes → new key → fresh cache).

Cache is scoped to branch by default. PRs can read base branch cache but save to their own scope.

</details>

---

### Q11 🔴 How do you implement a Jenkins shared library for reusable pipeline steps?

<details><summary>Click to reveal answer</summary>

```groovy
// vars/deployToECS.groovy (in shared library repo)
def call(Map config) {
    def cluster = config.cluster ?: 'default-cluster'
    def service = config.service
    def image = config.image
    def region = config.region ?: 'us-east-1'

    if (!service || !image) {
        error "deployToECS requires 'service' and 'image' parameters"
    }

    withAWS(region: region, credentials: config.credentials ?: 'aws-role') {
        sh """
            # Get current task definition
            TASK_DEF=\$(aws ecs describe-task-definition \
                --task-definition ${service} \
                --query taskDefinition)

            # Update image
            NEW_TASK=\$(echo \$TASK_DEF | jq \
                --arg IMG "${image}" \
                '.containerDefinitions[0].image = \$IMG | del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)')

            # Register new task definition
            NEW_ARN=\$(aws ecs register-task-definition \
                --cli-input-json "\$NEW_TASK" \
                --query taskDefinition.taskDefinitionArn \
                --output text)

            # Update service
            aws ecs update-service \
                --cluster ${cluster} \
                --service ${service} \
                --task-definition \$NEW_ARN \
                --force-new-deployment

            aws ecs wait services-stable \
                --cluster ${cluster} \
                --services ${service}
        """
    }
}
```

```groovy
// Jenkinsfile — using shared library
@Library('myorg-shared-library@main') _

pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployToECS(
                    cluster: 'myapp-cluster',
                    service: 'myapp-prod',
                    image: "123.dkr.ecr.us-east-1.amazonaws.com/myapp:${env.BUILD_NUMBER}",
                    credentials: 'aws-jenkins-role'
                )
            }
        }
    }
}
```

Configure in Jenkins: Manage Jenkins → System → Global Pipeline Libraries.

</details>

---

### Q12 🟡 How do you implement conditional job execution in GitHub Actions?

<details><summary>Click to reveal answer</summary>

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      is-release: ${{ steps.check.outputs.is-release }}

    steps:
      - id: check
        run: |
          if [[ "${{ github.ref }}" == refs/tags/v* ]]; then
            echo "is-release=true" >> $GITHUB_OUTPUT
          fi

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'    # only on main branch
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to staging"

  deploy-production:
    needs: [build, deploy-staging]
    # Use output from previous job
    if: needs.build.outputs.is-release == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to production"

  notify-failure:
    needs: [build, deploy-staging, deploy-production]
    if: failure()    # runs only if any previous job failed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Send failure notification"

  always-notify:
    needs: deploy-production
    if: always()     # runs regardless of previous job status
    runs-on: ubuntu-latest
    steps:
      - run: echo "Always runs"
```

</details>

---

### Q13 🔴 Explain how to implement a multi-environment promotion pipeline.

<details><summary>Click to reveal answer</summary>

```yaml
# .github/workflows/promote.yml
name: Multi-Environment Promotion

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.tag.outputs.tag }}
    steps:
      - id: tag
        run: echo "tag=sha-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
      - run: |
          docker build -t myapp:${{ steps.tag.outputs.tag }} .
          docker push 123.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ steps.tag.outputs.tag }}

  deploy-dev:
    needs: build
    environment: development
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/myapp myapp=myapp:${{ needs.build.outputs.image-tag }} -n dev

  deploy-staging:
    needs: [build, deploy-dev]
    environment: staging     # no required reviewers
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/myapp myapp=myapp:${{ needs.build.outputs.image-tag }} -n staging

  integration-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - run: mvn verify -Pintegration -Dapp.url=https://staging.myapp.com

  deploy-production:
    needs: [build, integration-tests]
    environment: production   # required reviewers configured in GitHub
    runs-on: ubuntu-latest
    steps:
      - run: kubectl set image deployment/myapp myapp=myapp:${{ needs.build.outputs.image-tag }} -n production
      - run: kubectl rollout status deployment/myapp -n production --timeout=5m
```

The same image tag flows through all environments — you're promoting the exact same artifact, not rebuilding.

</details>

---

### Q14 🟢 What is a Dockerfile multi-stage build and why is it used in CI/CD?

<details><summary>Click to reveal answer</summary>

```dockerfile
# Stage 1: Build (has Maven, JDK — large image)
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY pom.xml .
RUN mvn dependency:go-offline -q    # cache deps in layer
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: Runtime (only JRE — small image)
FROM eclipse-temurin:17-jre-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

**Benefits:**
- Final image has NO Maven, NO JDK — only JRE (~200MB vs ~600MB)
- Build artifacts stay in intermediate stage
- Security: no build tools in production image
- No need to build JAR separately in CI — `docker build` does everything

```yaml
# CI: single command builds and produces slim image
- run: docker build -t myapp:latest .
```

</details>

---

### Q15 🟡 How do you implement rollback in GitHub Actions?

<details><summary>Click to reveal answer</summary>

```yaml
# Manual rollback workflow
name: Rollback Production

on:
  workflow_dispatch:
    inputs:
      revision:
        description: 'Kubernetes revision number to rollback to (leave empty for previous)'
        required: false

jobs:
  rollback:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name myapp-cluster --region us-east-1

      - name: Rollback Kubernetes deployment
        run: |
          if [ -n "${{ inputs.revision }}" ]; then
            kubectl rollout undo deployment/myapp \
              --to-revision=${{ inputs.revision }} -n production
          else
            kubectl rollout undo deployment/myapp -n production
          fi

      - name: Wait for rollback to complete
        run: kubectl rollout status deployment/myapp -n production --timeout=5m

      - name: Verify health after rollback
        run: |
          sleep 15
          curl -f https://api.myapp.com/actuator/health
```

For ECS, keep previous task definition ARN and update service back to it:

```bash
# ECS rollback
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-service \
  --task-definition myapp-task:PREVIOUS_REVISION

aws ecs wait services-stable \
  --cluster myapp-cluster \
  --services myapp-service
```

</details>

---

*End of Chapter 4.4 — CI/CD Pipelines*
