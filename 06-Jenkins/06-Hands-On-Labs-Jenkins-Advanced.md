# Session 2 - Jenkins Advanced
## Full CI/CD Pipeline: Docker -> ECR -> ECS & Shared Libraries

---

## Prerequisites

> **Before this session:** Complete Session 1 labs. You should have Jenkins running with credentials configured, and a working Jenkinsfile pipeline that builds and tests the Flask app.

### AWS Setup (Required for this session)

```bash
# Configure AWS CLI
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name: us-east-1
# Default output format: json

# Verify access
aws sts get-caller-identity
```

**Pre-created AWS Resources:**
- ECS Cluster: `cicd-training-cluster`
- ECR Repository: `cicd-lab-app`
- IAM Role for ECS Task Execution: `ecsTaskExecRole-lab`
- VPC with public subnets and security groups configured
- ALB and Target Group configured

---

## ═════════════════════════════════════════════════════════
## Lab 1 - Full CI/CD Pipeline: Docker -> ECR -> ECS Deploy
## ═════════════════════════════════════════════════════════

**Objective:** Build a production-grade Jenkins pipeline that builds a Docker image, runs security scans, pushes to ECR, and deploys to an existing ECS cluster.

**Pre-requisite:** ECS cluster (`cicd-training-cluster`) must be running with a service already created. Refer the ECS notes for creating this in advance

### What You'll Learn
- ECR authentication inside a Jenkins pipeline
- Tagging strategy (build number + git SHA)
- ECS task definition update and service deploy
- Deployment verification (wait for service stability)
- Rollback strategy

---

### Step 1: Store AWS Credentials in Jenkins

Go to **Manage Jenkins -> Credentials -> Global -> Add Credentials:**

```
Kind: AWS Credentials
ID: aws-ecr-credentials
Description: AWS ECR + ECS Access
Access Key ID: <your key>
Secret Access Key: <your secret>
```

Also add:
```
Kind: Secret text
ID: ecr-repo-uri
Description: ECR Repository URI
Secret: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/cicd-lab-app
```

### Step 2: Create the Production Jenkinsfile

Create `Jenkinsfile.prod` in your repo:

```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        AWS_REGION         = 'us-east-1'
        ECR_REPO_URI       = credentials('ecr-repo-uri')
        ECS_CLUSTER        = 'cicd-training-cluster'
        ECS_SERVICE        = 'lab-app-service'
        TASK_FAMILY        = 'lab-app-task'
        CONTAINER_NAME     = 'lab-app'
        IMAGE_TAG          = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(8)}"
    }

    stages {
        stage('Checkout & Validate') {
            steps {
                script {
                    echo "============================================"
                    echo " CI/CD PIPELINE: ${env.JOB_NAME}"
                    echo " Build:   #${env.BUILD_NUMBER}"
                    echo " Branch:  ${env.GIT_BRANCH}"
                    echo " Commit:  ${env.GIT_COMMIT?.take(8)}"
                    echo " Image:   ${env.IMAGE_TAG}"
                    echo "============================================"
                }
                // Ensure Dockerfile exists
                sh 'test -f Dockerfile || (echo "Dockerfile not found!" && exit 1)'
                sh 'test -f requirements.txt || (echo "requirements.txt not found!" && exit 1)'
            }
        }

        stage('Install & Test') {
            agent {
                docker {
                    image 'python:3.11-slim'
                    reuseNode true
                }
            }
            steps {
                sh 'pip install -r requirements.txt --quiet'
                sh 'python -m pytest tests/ -v --junitxml=test-results.xml'
            }
            post {
                always { junit 'test-results.xml' }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build \
                            --build-arg APP_VERSION=${env.IMAGE_TAG} \
                            --label "git.commit=${env.GIT_COMMIT}" \
                            --label "build.number=${env.BUILD_NUMBER}" \
                            --label "build.date=\$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            -t ${env.CONTAINER_NAME}:${env.IMAGE_TAG} \
                            .
                    """
                    echo "Built: ${env.CONTAINER_NAME}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh '''
                    # Check if trivy is installed, install if not
                    command -v trivy || {
                        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                          | sh -s -- -b /usr/local/bin latest
                    }
                    
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        --format table \
                        ''' + env.CONTAINER_NAME + ':' + env.IMAGE_TAG + '''
                    
                    echo "Security scan complete"
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                withAWS(region: "${env.AWS_REGION}", credentials: 'aws-ecr-credentials') {
                    script {
                        // Get account ID
                        def accountId = sh(
                            script: 'aws sts get-caller-identity --query Account --output text',
                            returnStdout: true
                        ).trim()

                        // Authenticate Docker to ECR
                        sh """
                            aws ecr get-login-password --region ${env.AWS_REGION} | \
                                docker login --username AWS \
                                --password-stdin ${accountId}.dkr.ecr.${env.AWS_REGION}.amazonaws.com
                        """

                        // Tag for ECR
                        sh """
                            docker tag ${env.CONTAINER_NAME}:${env.IMAGE_TAG} \
                                ${env.ECR_REPO_URI}:${env.IMAGE_TAG}
                            docker tag ${env.CONTAINER_NAME}:${env.IMAGE_TAG} \
                                ${env.ECR_REPO_URI}:latest
                        """

                        // Push both tags
                        sh """
                            docker push ${env.ECR_REPO_URI}:${env.IMAGE_TAG}
                            docker push ${env.ECR_REPO_URI}:latest
                        """

                        echo "Pushed: ${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                        echo "Pushed: ${env.ECR_REPO_URI}:latest"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            when {
                branch 'main'
            }
            steps {
                withAWS(region: "${env.AWS_REGION}", credentials: 'aws-ecr-credentials') {
                    script {
                        echo "Fetching current task definition..."
                        
                        // Get current task definition
                        def taskDefJson = sh(
                            script: """
                                aws ecs describe-task-definition \
                                    --task-definition ${env.TASK_FAMILY} \
                                    --query 'taskDefinition' \
                                    --output json
                            """,
                            returnStdout: true
                        ).trim()

                        // Update image in task definition
                        def updatedTaskDef = sh(
                            script: """
                                echo '${taskDefJson}' | python3 -c "
import json, sys
td = json.load(sys.stdin)
for cd in td['containerDefinitions']:
    if cd['name'] == '${env.CONTAINER_NAME}':
        cd['image'] = '${env.ECR_REPO_URI}:${env.IMAGE_TAG}'
# Remove fields that can't be in registration request
for field in ['taskDefinitionArn','revision','status','requiresAttributes','compatibilities','registeredAt','registeredBy']:
    td.pop(field, None)
print(json.dumps(td))
"
                            """,
                            returnStdout: true
                        ).trim()

                        // Register new task definition
                        def newTaskDefArn = sh(
                            script: """
                                echo '${updatedTaskDef}' | aws ecs register-task-definition \
                                    --cli-input-json file:///dev/stdin \
                                    --query 'taskDefinition.taskDefinitionArn' \
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()

                        echo "New task definition: ${newTaskDefArn}"

                        // Update ECS service
                        sh """
                            aws ecs update-service \
                                --cluster ${env.ECS_CLUSTER} \
                                --service ${env.ECS_SERVICE} \
                                --task-definition ${newTaskDefArn} \
                                --force-new-deployment \
                                --region ${env.AWS_REGION}
                        """

                        echo "Deployment triggered. Waiting for service to stabilize..."

                        // Wait for service to be stable (max 10 minutes)
                        sh """
                            aws ecs wait services-stable \
                                --cluster ${env.ECS_CLUSTER} \
                                --services ${env.ECS_SERVICE} \
                                --region ${env.AWS_REGION}
                        """

                        echo "Deployment complete! Service is stable."
                    }
                }
            }
        }

        stage('Deployment Verification') {
            when {
                branch 'main'
            }
            steps {
                withAWS(region: "${env.AWS_REGION}", credentials: 'aws-ecr-credentials') {
                    script {
                        // Get running tasks count
                        def runningCount = sh(
                            script: """
                                aws ecs describe-services \
                                    --cluster ${env.ECS_CLUSTER} \
                                    --services ${env.ECS_SERVICE} \
                                    --query 'services[0].runningCount' \
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Running tasks: ${runningCount}"

                        if (runningCount.toInteger() == 0) {
                            error("Deployment failed - no running tasks!")
                        }

                        echo "Verification passed! ${runningCount} tasks running."
                    }
                }
            }
        }
    }

    post {
        always {
            // Clean up local Docker images to save disk
            sh '''
                docker rmi $(docker images -q --filter "dangling=true") 2>/dev/null || true
            '''
        }
        success {
            echo "DEPLOYED SUCCESSFULLY: ${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
        }
        failure {
            echo "DEPLOYMENT FAILED - Image: ${env.IMAGE_TAG}"
            // Rollback hint
            echo "To rollback: aws ecs update-service --cluster ${env.ECS_CLUSTER} --service ${env.ECS_SERVICE} --task-definition <PREVIOUS_TASK_DEF>"
        }
    }
}
```

### Step 3: Create the Pipeline Job

1. **New Item** -> `lab-cicd-pipeline` -> **Pipeline** -> OK
2. Pipeline -> SCM -> Git
3. Repository: your repo
4. Branch: `*/main`
5. Script Path: `Jenkinsfile.prod`
6. **Save** -> **Build Now**

### Step 4: Monitor and Verify Deployment

```bash
# Watch ECS service update in real-time
watch -n 5 "aws ecs describe-services \
    --cluster cicd-training-cluster \
    --services lab-app-service \
    --query 'services[0].{Running:runningCount,Desired:desiredCount,Pending:pendingCount}' \
    --output table"

# Verify the new image is running
aws ecs describe-tasks \
    --cluster cicd-training-cluster \
    --tasks $(aws ecs list-tasks --cluster cicd-training-cluster --service-name lab-app-service --query 'taskArns[0]' --output text) \
    --query 'tasks[0].containers[0].image'
```

### ✅ Lab 1 Success Criteria
- Pipeline runs all 7 stages successfully
- Docker image tagged with build-number + git SHA
- Image visible in ECR with correct tags
- ECS service updated with new task definition
- `aws ecs wait services-stable` completes without error

---

## ════════════════════════════════════════════════
## Lab 2 - Jenkins Shared Libraries (DRY Pipelines)
## ════════════════════════════════════════════════

**Objective:** Create a Jenkins Shared Library that encapsulates Docker build, ECR push, and ECS deploy logic. Consume it from a slim Jenkinsfile.

### What You'll Learn
- Shared Library directory structure
- Writing `vars/` global variables (DSL steps)
- Loading libraries in Jenkinsfiles with `@Library`
- The DRY principle at pipeline scale

---

### Step 1: Create the Shared Library Repository

Create a new GitHub repo: `jenkins-shared-lib`

```bash
mkdir jenkins-shared-lib && cd jenkins-shared-lib
git init

mkdir -p vars src/com/company resources
```

**`vars/dockerBuild.groovy`:**
```groovy
def call(Map config = [:]) {
    def imageName  = config.imageName  ?: error("dockerBuild: imageName is required")
    def imageTag   = config.imageTag   ?: env.BUILD_NUMBER
    def dockerfile = config.dockerfile ?: 'Dockerfile'
    def buildArgs  = config.buildArgs  ?: ''

    sh """
        docker build \
            ${buildArgs} \
            --label "git.commit=${env.GIT_COMMIT}" \
            --label "build.number=${env.BUILD_NUMBER}" \
            -f ${dockerfile} \
            -t ${imageName}:${imageTag} \
            -t ${imageName}:latest \
            .
    """
    echo "Built: ${imageName}:${imageTag}"
    return "${imageName}:${imageTag}"
}
```

**`vars/ecrPush.groovy`:**
```groovy
def call(Map config = [:]) {
    def imageName    = config.imageName    ?: error("ecrPush: imageName is required")
    def imageTag     = config.imageTag     ?: env.BUILD_NUMBER
    def ecrRepoUri   = config.ecrRepoUri   ?: error("ecrPush: ecrRepoUri is required")
    def awsRegion    = config.awsRegion    ?: 'us-east-1'
    def awsCredId    = config.awsCredId    ?: 'aws-ecr-credentials'

    withAWS(region: awsRegion, credentials: awsCredId) {
        def registryUrl = ecrRepoUri.split('/')[0]

        sh """
            aws ecr get-login-password --region ${awsRegion} | \
                docker login --username AWS --password-stdin ${registryUrl}

            docker tag ${imageName}:${imageTag} ${ecrRepoUri}:${imageTag}
            docker tag ${imageName}:${imageTag} ${ecrRepoUri}:latest

            docker push ${ecrRepoUri}:${imageTag}
            docker push ${ecrRepoUri}:latest
        """
        echo "Pushed: ${ecrRepoUri}:${imageTag}"
    }
}
```

**`vars/ecsDeploy.groovy`:**
```groovy
def call(Map config = [:]) {
    def cluster      = config.cluster      ?: error("ecsDeploy: cluster is required")
    def service      = config.service      ?: error("ecsDeploy: service is required")
    def taskFamily   = config.taskFamily   ?: error("ecsDeploy: taskFamily is required")
    def containerName = config.containerName ?: 'app'
    def imageUri     = config.imageUri     ?: error("ecsDeploy: imageUri is required")
    def awsRegion    = config.awsRegion    ?: 'us-east-1'
    def awsCredId    = config.awsCredId    ?: 'aws-ecr-credentials'
    def waitStable   = config.waitStable   != null ? config.waitStable : true

    withAWS(region: awsRegion, credentials: awsCredId) {
        script {
            // Update task definition with new image
            def taskDefJson = sh(
                script: "aws ecs describe-task-definition --task-definition ${taskFamily} --query taskDefinition --output json",
                returnStdout: true
            ).trim()

            def newTaskDef = sh(
                script: """
                    echo '${taskDefJson}' | python3 -c "
import json, sys
td = json.load(sys.stdin)
for cd in td['containerDefinitions']:
    if cd['name'] == '${containerName}':
        cd['image'] = '${imageUri}'
for f in ['taskDefinitionArn','revision','status','requiresAttributes','compatibilities','registeredAt','registeredBy']:
    td.pop(f, None)
print(json.dumps(td))
"
                """,
                returnStdout: true
            ).trim()

            def newArn = sh(
                script: "echo '${newTaskDef}' | aws ecs register-task-definition --cli-input-json file:///dev/stdin --query taskDefinition.taskDefinitionArn --output text",
                returnStdout: true
            ).trim()

            sh """
                aws ecs update-service \
                    --cluster ${cluster} \
                    --service ${service} \
                    --task-definition ${newArn} \
                    --force-new-deployment
            """

            if (waitStable) {
                echo "Waiting for service to stabilize..."
                sh "aws ecs wait services-stable --cluster ${cluster} --services ${service}"
                echo "Deployment stable!"
            }
        }
    }
}
```

**`vars/trivyScan.groovy`:**
```groovy
def call(Map config = [:]) {
    def imageName  = config.imageName  ?: error("trivyScan: imageName is required")
    def imageTag   = config.imageTag   ?: 'latest'
    def severity   = config.severity   ?: 'HIGH,CRITICAL'
    def failBuild  = config.failBuild  != null ? config.failBuild : false

    sh """
        command -v trivy || {
            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
                | sh -s -- -b /usr/local/bin latest
        }
        trivy image \
            --severity ${severity} \
            --exit-code ${failBuild ? 1 : 0} \
            --no-progress \
            --format table \
            ${imageName}:${imageTag}
    """
}
```

**`vars/runTests.groovy`:**
```groovy
def call(Map config = [:]) {
    def pythonImage = config.pythonImage ?: 'python:3.11-slim'
    def testCommand = config.testCommand ?: 'python -m pytest tests/ -v --junitxml=test-results.xml'

    docker.image(pythonImage).inside {
        sh 'pip install -r requirements.txt --quiet'
        sh testCommand
    }
    junit 'test-results.xml'
}
```

```bash
cd jenkins-shared-lib
git add .
git commit -m "Initial shared library"
git remote add origin https://github.com/YOUR_USERNAME/jenkins-shared-lib.git
git push -u origin main
```

### Step 2: Register the Shared Library in Jenkins

1. **Manage Jenkins -> System -> Global Pipeline Libraries**
2. Click **Add**:
```
Name: jenkins-shared-lib
Default version: main
Retrieval method: Modern SCM -> Git
Project repository: https://github.com/YOUR_USERNAME/jenkins-shared-lib.git
Credentials: github-token
```
3. **Save**

### Step 3: Consume the Library in a Slim Jenkinsfile

Create `Jenkinsfile.shared-lib` in your app repo:

```groovy
@Library('jenkins-shared-lib@main') _

pipeline {
    agent any

    environment {
        ECR_REPO_URI = credentials('ecr-repo-uri')
        IMAGE_NAME   = 'cicd-lab-app'
        IMAGE_TAG    = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(8)}"
        ECS_CLUSTER  = 'cicd-training-cluster'
        ECS_SERVICE  = 'lab-app-service'
    }

    stages {
        stage('Test') {
            steps {
                runTests(pythonImage: 'python:3.11-slim')
            }
        }

        stage('Build') {
            steps {
                dockerBuild(imageName: env.IMAGE_NAME, imageTag: env.IMAGE_TAG)
            }
        }

        stage('Scan') {
            steps {
                trivyScan(imageName: env.IMAGE_NAME, imageTag: env.IMAGE_TAG, failBuild: false)
            }
        }

        stage('Push') {
            steps {
                ecrPush(
                    imageName:  env.IMAGE_NAME,
                    imageTag:   env.IMAGE_TAG,
                    ecrRepoUri: env.ECR_REPO_URI
                )
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                ecsDeploy(
                    cluster:       env.ECS_CLUSTER,
                    service:       env.ECS_SERVICE,
                    taskFamily:    'lab-app-task',
                    containerName: 'lab-app',
                    imageUri:      "${env.ECR_REPO_URI}:${env.IMAGE_TAG}"
                )
            }
        }
    }
}
```

Notice how the full pipeline is now ~50 lines of clean, readable Groovy. The complexity lives in the shared library, tested once and used everywhere.

### Step 4: Test a Library Change

1. Update `vars/trivyScan.groovy` to add `--format json` output
2. Push to `jenkins-shared-lib`
3. Re-run the pipeline - all pipelines using this library get the update automatically

### ✅ Lab 2 Success Criteria
- Shared library repo created with 5 `vars/` files
- Library registered in Jenkins global settings
- Slim Jenkinsfile runs successfully using library functions
- Console shows the library being loaded from GitHub
- Test a library update propagates to all consuming pipelines

---