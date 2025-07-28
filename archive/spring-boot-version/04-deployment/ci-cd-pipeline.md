# CI/CD Pipeline Guide

This document provides comprehensive instructions for setting up and maintaining CI/CD pipelines for the First Viscount microservices platform.

## Overview

The CI/CD pipeline automates:
- Code quality checks
- Unit and integration testing
- Security scanning
- Container image building
- Deployment to multiple environments
- Performance testing
- Rollback procedures

## Pipeline Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│   Source    │────▶│    Build     │────▶│    Test     │────▶│   Deploy     │
│    Code     │     │   & Scan     │     │  & Verify   │     │ & Monitor    │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────────┘
      │                    │                     │                    │
      ▼                    ▼                     ▼                    ▼
  Git Push            Compile              Unit Tests          Deploy to Dev
  PR Created          Lint/Format         Integration Tests    Deploy to Staging
  Tag Created         Security Scan       Contract Tests       Deploy to Prod
                      Build Images         Performance Tests   Monitor & Alert
```

## GitHub Actions Pipeline

### Main Workflow

```yaml
# .github/workflows/main.yml
name: First Viscount CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository }}
  JAVA_VERSION: '21'
  NODE_VERSION: '18'

jobs:
  # Code Quality Checks
  quality-check:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for SonarQube

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Cache SonarQube packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar

      - name: Run Checkstyle
        run: mvn checkstyle:check

      - name: Run PMD
        run: mvn pmd:check

      - name: Run SpotBugs
        run: mvn spotbugs:check

      - name: SonarQube Scan
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn clean verify sonar:sonar \
            -Dsonar.projectKey=firstviscount \
            -Dsonar.organization=firstviscount \
            -Dsonar.host.url=https://sonarcloud.io

  # Security Scanning
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'firstviscount'
          path: '.'
          format: 'ALL'
          args: >
            --enableRetired
            --enableExperimental

      - name: Upload OWASP results
        uses: actions/upload-artifact@v3
        with:
          name: owasp-report
          path: reports/

  # Build and Test Matrix
  build-test:
    name: Build and Test - ${{ matrix.service }}
    runs-on: ubuntu-latest
    needs: [quality-check]
    strategy:
      matrix:
        service:
          - platform-coordination
          - product-service
          - inventory-service
          - order-service
          - delivery-service
          - notification-service
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      kafka:
        image: confluentinc/cp-kafka:latest
        ports:
          - 9092:9092
        env:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
        options: >-
          --health-cmd "kafka-topics --bootstrap-server localhost:9092 --list"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Run Unit Tests
        working-directory: ./${{ matrix.service }}
        run: |
          mvn clean test \
            -Dspring.profiles.active=test \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

      - name: Run Integration Tests
        working-directory: ./${{ matrix.service }}
        run: |
          mvn verify -P integration-test \
            -Dspring.profiles.active=integration-test

      - name: Generate Test Report
        if: always()
        working-directory: ./${{ matrix.service }}
        run: mvn surefire-report:report

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.service }}
          path: |
            ${{ matrix.service }}/target/surefire-reports/
            ${{ matrix.service }}/target/site/jacoco/

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./${{ matrix.service }}/target/site/jacoco/jacoco.xml
          flags: ${{ matrix.service }}
          name: ${{ matrix.service }}-coverage

  # Contract Testing
  contract-test:
    name: Contract Testing
    runs-on: ubuntu-latest
    needs: [build-test]
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      - name: Run Contract Tests
        run: |
          mvn clean test -P contract-tests \
            -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }} \
            -Dpact.broker.token=${{ secrets.PACT_BROKER_TOKEN }}

      - name: Publish Pacts
        if: success()
        run: |
          mvn pact:publish \
            -Dpact.broker.url=${{ secrets.PACT_BROKER_URL }} \
            -Dpact.broker.token=${{ secrets.PACT_BROKER_TOKEN }} \
            -Dpact.consumer.version=${{ github.sha }}

  # Build Docker Images
  build-images:
    name: Build Images - ${{ matrix.service }}
    runs-on: ubuntu-latest
    needs: [build-test, security-scan]
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    strategy:
      matrix:
        service:
          - platform-coordination
          - product-service
          - inventory-service
          - order-service
          - delivery-service
          - notification-service
    
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      - name: Build JAR
        working-directory: ./${{ matrix.service }}
        run: mvn clean package -DskipTests

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            JAR_FILE=target/*.jar
            JAVA_VERSION=${{ env.JAVA_VERSION }}

  # Performance Testing
  performance-test:
    name: Performance Testing
    runs-on: ubuntu-latest
    needs: [build-images]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up K6
        run: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6

      - name: Deploy Test Environment
        run: |
          docker-compose -f docker-compose.test.yml up -d
          ./scripts/wait-for-services.sh

      - name: Run Performance Tests
        run: |
          k6 run \
            --out cloud \
            --out json=performance-results.json \
            performance-tests/load-test.js

      - name: Upload Performance Results
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: performance-results.json

      - name: Check Performance Thresholds
        run: |
          python scripts/check-performance.py performance-results.json

  # Deploy to Development
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [build-images]
    if: github.ref == 'refs/heads/develop'
    environment: development
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Set up Kubeconfig
        run: |
          echo "${{ secrets.DEV_KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=$(pwd)/kubeconfig

      - name: Deploy to Development
        run: |
          ./scripts/deploy.sh development ${{ github.sha }}

      - name: Wait for Deployment
        run: |
          kubectl wait --for=condition=available --timeout=300s \
            deployment --all -n firstviscount-dev

      - name: Run Smoke Tests
        run: |
          npm install -g newman
          newman run postman/smoke-tests.json \
            -e postman/dev-environment.json

  # Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [deploy-dev]
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Set up Kubeconfig
        run: |
          echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=$(pwd)/kubeconfig

      - name: Deploy to Staging
        run: |
          ./scripts/deploy.sh staging ${{ github.sha }}

      - name: Run E2E Tests
        run: |
          npm install -g newman
          newman run postman/e2e-tests.json \
            -e postman/staging-environment.json \
            --timeout-request 30000

      - name: Security Scan Production Images
        run: |
          for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
            trivy image --severity HIGH,CRITICAL \
              ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/$service:${{ github.sha }}
          done

  # Deploy to Production
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment: production
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Set up Kubeconfig
        run: |
          echo "${{ secrets.PROD_KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=$(pwd)/kubeconfig

      - name: Create Backup
        run: |
          ./scripts/backup-prod.sh

      - name: Deploy to Production
        run: |
          ./scripts/deploy.sh production ${{ github.ref_name }}

      - name: Monitor Deployment
        run: |
          ./scripts/monitor-deployment.sh production

      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref_name }}
          release_name: Release ${{ github.ref_name }}
          body: |
            Release ${{ github.ref_name }}
            
            Changes in this release:
            - See [CHANGELOG.md](https://github.com/${{ github.repository }}/blob/main/CHANGELOG.md)
            
            Docker Images:
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/platform-coordination:${{ github.ref_name }}`
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/product-service:${{ github.ref_name }}`
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/inventory-service:${{ github.ref_name }}`
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/order-service:${{ github.ref_name }}`
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/delivery-service:${{ github.ref_name }}`
            - `${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/notification-service:${{ github.ref_name }}`
          draft: false
          prerelease: false

  # Rollback Job
  rollback:
    name: Rollback Deployment
    runs-on: ubuntu-latest
    if: failure() && (needs.deploy-staging.result == 'failure' || needs.deploy-prod.result == 'failure')
    needs: [deploy-staging, deploy-prod]
    
    steps:
      - uses: actions/checkout@v4

      - name: Determine Environment
        id: env
        run: |
          if [[ "${{ needs.deploy-prod.result }}" == "failure" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "kubeconfig=${{ secrets.PROD_KUBECONFIG }}" >> $GITHUB_OUTPUT
          else
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "kubeconfig=${{ secrets.STAGING_KUBECONFIG }}" >> $GITHUB_OUTPUT
          fi

      - name: Configure kubectl
        run: |
          echo "${{ steps.env.outputs.kubeconfig }}" | base64 -d > kubeconfig
          export KUBECONFIG=$(pwd)/kubeconfig

      - name: Rollback Deployment
        run: |
          ./scripts/rollback.sh ${{ steps.env.outputs.environment }}

      - name: Notify Team
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              "text": "Deployment Rollback",
              "attachments": [{
                "color": "warning",
                "fields": [{
                  "title": "Environment",
                  "value": "${{ steps.env.outputs.environment }}",
                  "short": true
                }, {
                  "title": "Triggered by",
                  "value": "${{ github.actor }}",
                  "short": true
                }]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Service-Specific Workflow

```yaml
# .github/workflows/service-build.yml
name: Service Build

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      java-version:
        required: false
        type: string
        default: '21'
    secrets:
      SONAR_TOKEN:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ inputs.java-version }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Build with Maven
        working-directory: ./${{ inputs.service-name }}
        run: |
          mvn clean compile

      - name: Run Tests
        working-directory: ./${{ inputs.service-name }}
        run: |
          mvn test

      - name: SonarQube Analysis
        working-directory: ./${{ inputs.service-name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=${{ inputs.service-name }} \
            -Dsonar.organization=firstviscount
```

## GitLab CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - quality
  - security
  - build
  - test
  - package
  - deploy
  - monitor

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  REGISTRY: $CI_REGISTRY
  JAVA_VERSION: "21"

cache:
  paths:
    - .m2/repository
    - target/

# Templates
.java-build:
  image: maven:3.9-eclipse-temurin-21
  before_script:
    - cd $SERVICE_NAME

.docker-build:
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# Quality Stage
code-quality:
  stage: quality
  extends: .java-build
  parallel:
    matrix:
      - SERVICE_NAME: [platform-coordination, product-service, inventory-service, order-service, delivery-service, notification-service]
  script:
    - mvn clean compile
    - mvn checkstyle:check
    - mvn pmd:check
    - mvn spotbugs:check
  artifacts:
    reports:
      junit:
        - "**/target/surefire-reports/TEST-*.xml"

sonarqube:
  stage: quality
  extends: .java-build
  script:
    - mvn verify sonar:sonar -Dsonar.qualitygate.wait=true
  only:
    - merge_requests
    - main
    - develop

# Security Stage
dependency-check:
  stage: security
  image: owasp/dependency-check-action:latest
  script:
    - dependency-check.sh --project "FirstViscount" --scan . --format ALL
  artifacts:
    reports:
      junit: dependency-check-report.xml
    paths:
      - dependency-check-report.*

container-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy fs --security-checks vuln,config --severity HIGH,CRITICAL .
  allow_failure: true

# Build Stage
build:maven:
  stage: build
  extends: .java-build
  parallel:
    matrix:
      - SERVICE_NAME: [platform-coordination, product-service, inventory-service, order-service, delivery-service, notification-service]
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - "*/target/*.jar"
    expire_in: 1 week

# Test Stage
unit-tests:
  stage: test
  extends: .java-build
  needs: ["build:maven"]
  parallel:
    matrix:
      - SERVICE_NAME: [platform-coordination, product-service, inventory-service, order-service, delivery-service, notification-service]
  script:
    - mvn test
  artifacts:
    reports:
      junit:
        - "*/target/surefire-reports/TEST-*.xml"
      coverage_report:
        coverage_format: cobertura
        path: "*/target/site/cobertura/coverage.xml"

integration-tests:
  stage: test
  extends: .java-build
  needs: ["unit-tests"]
  services:
    - postgres:15-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    SPRING_PROFILES_ACTIVE: integration-test
  parallel:
    matrix:
      - SERVICE_NAME: [platform-coordination, product-service, inventory-service, order-service, delivery-service, notification-service]
  script:
    - mvn verify -P integration-test

contract-tests:
  stage: test
  extends: .java-build
  needs: ["unit-tests"]
  script:
    - mvn test -P contract-tests
    - mvn pact:publish -Dpact.broker.url=$PACT_BROKER_URL

# Package Stage
build:docker:
  stage: package
  extends: .docker-build
  needs: ["integration-tests"]
  parallel:
    matrix:
      - SERVICE_NAME: [platform-coordination, product-service, inventory-service, order-service, delivery-service, notification-service]
  script:
    - cd $SERVICE_NAME
    - docker build -t $REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:$CI_COMMIT_SHA .
    - docker tag $REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:$CI_COMMIT_SHA $REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:latest
    - docker push $REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:$CI_COMMIT_SHA
    - docker push $REGISTRY/$CI_PROJECT_PATH/$SERVICE_NAME:latest

# Deploy Stage
deploy:dev:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["build:docker"]
  environment:
    name: development
    url: https://dev.firstviscount.com
  script:
    - kubectl config use-context dev
    - ./scripts/deploy.sh development $CI_COMMIT_SHA
  only:
    - develop

deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["build:docker", "deploy:dev"]
  environment:
    name: staging
    url: https://staging.firstviscount.com
  script:
    - kubectl config use-context staging
    - ./scripts/deploy.sh staging $CI_COMMIT_SHA
  only:
    - main

deploy:prod:
  stage: deploy
  image: bitnami/kubectl:latest
  needs: ["deploy:staging"]
  environment:
    name: production
    url: https://api.firstviscount.com
  script:
    - kubectl config use-context prod
    - ./scripts/backup-prod.sh
    - ./scripts/deploy.sh production $CI_TAG
  only:
    - tags
  when: manual

# Monitor Stage
smoke-tests:
  stage: monitor
  image: postman/newman:alpine
  needs: ["deploy:dev"]
  script:
    - newman run postman/smoke-tests.json -e postman/$CI_ENVIRONMENT_NAME-environment.json
  only:
    - develop
    - main

performance-tests:
  stage: monitor
  image: loadimpact/k6:latest
  needs: ["deploy:staging"]
  script:
    - k6 run --out cloud performance-tests/load-test.js
  only:
    - main
  when: manual
```

## Jenkins Pipeline

```groovy
// Jenkinsfile
@Library('jenkins-shared-library') _

pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-21
    command: ['sleep', '99999']
  - name: docker
    image: docker:latest
    command: ['sleep', '99999']
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['sleep', '99999']
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
'''
        }
    }

    environment {
        REGISTRY = 'registry.firstviscount.com'
        REGISTRY_CREDENTIALS = credentials('docker-registry')
        SONAR_TOKEN = credentials('sonar-token')
        SERVICES = 'platform-coordination,product-service,inventory-service,order-service,delivery-service,notification-service'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    env.GIT_BRANCH = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()
                }
            }
        }

        stage('Quality Gates') {
            parallel {
                stage('Code Analysis') {
                    steps {
                        container('maven') {
                            sh '''
                                mvn clean compile
                                mvn checkstyle:check
                                mvn pmd:check
                                mvn spotbugs:check
                            '''
                        }
                    }
                }

                stage('Security Scan') {
                    steps {
                        container('maven') {
                            sh 'mvn dependency-check:check'
                        }
                    }
                }

                stage('SonarQube') {
                    steps {
                        container('maven') {
                            withSonarQubeEnv('SonarQube') {
                                sh 'mvn sonar:sonar'
                            }
                        }
                    }
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Test') {
            steps {
                container('maven') {
                    script {
                        def services = env.SERVICES.split(',')
                        parallel services.collectEntries { service ->
                            ["${service}": {
                                dir(service) {
                                    sh 'mvn clean package'
                                    junit '**/target/surefire-reports/*.xml'
                                    publishHTML([
                                        allowMissing: false,
                                        alwaysLinkToLastBuild: true,
                                        keepAll: true,
                                        reportDir: 'target/site/jacoco',
                                        reportFiles: 'index.html',
                                        reportName: "${service} Coverage Report"
                                    ])
                                }
                            }]
                        }
                    }
                }
            }
        }

        stage('Build Images') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                    tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
                }
            }
            steps {
                container('docker') {
                    script {
                        docker.withRegistry("https://${REGISTRY}", 'docker-registry') {
                            def services = env.SERVICES.split(',')
                            services.each { service ->
                                dir(service) {
                                    def image = docker.build("${REGISTRY}/firstviscount/${service}:${GIT_COMMIT}")
                                    image.push()
                                    if (env.GIT_BRANCH == 'main') {
                                        image.push('latest')
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'dev-kubeconfig']) {
                        sh "./scripts/deploy.sh development ${GIT_COMMIT}"
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                container('kubectl') {
                    withKubeConfig([credentialsId: 'staging-kubeconfig']) {
                        sh "./scripts/deploy.sh staging ${GIT_COMMIT}"
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                container('kubectl') {
                    withKubeConfig([credentialsId: 'prod-kubeconfig']) {
                        sh "./scripts/backup-prod.sh"
                        sh "./scripts/deploy.sh production ${TAG_NAME}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "Build Success: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)"
            )
        }
    }
}
```

## Deployment Scripts

### Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=$1
VERSION=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$VERSION" ]; then
    echo "Usage: ./deploy.sh <environment> <version>"
    exit 1
fi

echo "Deploying version $VERSION to $ENVIRONMENT..."

# Update image tags
for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
    kubectl set image deployment/$service $service=$REGISTRY/firstviscount/$service:$VERSION \
        -n firstviscount-$ENVIRONMENT
done

# Wait for rollout
for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
    kubectl rollout status deployment/$service -n firstviscount-$ENVIRONMENT
done

# Run post-deployment tests
./scripts/post-deploy-test.sh $ENVIRONMENT

echo "Deployment completed successfully!"
```

### Rollback Script

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

ENVIRONMENT=$1

if [ -z "$ENVIRONMENT" ]; then
    echo "Usage: ./rollback.sh <environment>"
    exit 1
fi

echo "Rolling back deployment in $ENVIRONMENT..."

# Rollback all deployments
for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
    kubectl rollout undo deployment/$service -n firstviscount-$ENVIRONMENT
done

# Wait for rollback
for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
    kubectl rollout status deployment/$service -n firstviscount-$ENVIRONMENT
done

echo "Rollback completed!"
```

### Performance Check Script

```python
#!/usr/bin/env python3
# scripts/check-performance.py

import json
import sys

def check_performance(results_file):
    with open(results_file, 'r') as f:
        results = json.load(f)
    
    thresholds = {
        'http_req_duration': {'p95': 500, 'p99': 1000},
        'http_req_failed': {'rate': 0.01},
        'iterations': {'rate': 10}
    }
    
    failed_checks = []
    
    for metric, limits in thresholds.items():
        if metric in results['metrics']:
            metric_data = results['metrics'][metric]
            for stat, limit in limits.items():
                if stat in metric_data and metric_data[stat] > limit:
                    failed_checks.append(f"{metric}.{stat}: {metric_data[stat]} > {limit}")
    
    if failed_checks:
        print("Performance thresholds exceeded:")
        for check in failed_checks:
            print(f"  - {check}")
        sys.exit(1)
    else:
        print("All performance thresholds passed!")

if __name__ == "__main__":
    check_performance(sys.argv[1])
```

## Build Configuration

### Multi-Stage Dockerfile

```dockerfile
# Base Dockerfile for all services
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

# Copy POM and download dependencies
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime image
FROM eclipse-temurin:21-jre-alpine

RUN addgroup -g 1000 app && adduser -u 1000 -G app -s /bin/sh -D app

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

RUN chown -R app:app /app

USER app

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Security Scanning

### Trivy Configuration

```yaml
# .trivyignore
# Ignore specific CVEs
CVE-2021-12345

# Ignore vulnerabilities in test dependencies
*/test/*
```

### OWASP Dependency Check

```xml
<!-- owasp-suppressions.xml -->
<suppressions>
    <suppress>
        <notes>False positive in test dependency</notes>
        <cve>CVE-2021-00000</cve>
    </suppress>
</suppressions>
```

## Monitoring and Alerts

### Deployment Monitoring

```yaml
# monitoring/deployment-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: deployment-alerts
  namespace: firstviscount-prod
spec:
  groups:
  - name: deployment
    interval: 30s
    rules:
    - alert: DeploymentFailed
      expr: |
        kube_deployment_status_condition{condition="Available",status="false"} == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Deployment {{ $labels.deployment }} is not available"
        description: "Deployment {{ $labels.deployment }} in namespace {{ $labels.namespace }} has been unavailable for more than 5 minutes"
    
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m]) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Service {{ $labels.service }} is experiencing {{ $value | humanizePercentage }} error rate"
```

## Best Practices

1. **Automated Testing**: Run comprehensive tests at every stage
2. **Security First**: Scan for vulnerabilities early and often
3. **Progressive Deployment**: Deploy to dev → staging → production
4. **Feature Flags**: Use feature flags for gradual rollouts
5. **Monitoring**: Monitor deployments and automatically rollback on failures
6. **Documentation**: Keep deployment documentation up to date
7. **Secrets Management**: Never commit secrets, use secure vaults
8. **Immutable Infrastructure**: Build once, deploy many times
9. **Rollback Strategy**: Always have a quick rollback plan
10. **Audit Trail**: Log all deployment activities

---
*Last Updated: January 2025*