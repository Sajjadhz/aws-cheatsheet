To achieve the same goal using AWS CLI, you need to break down the process into several steps. Below are the detailed steps along with the required AWS CLI commands and scripts:

### 1. Prerequisites

- **AWS CLI** installed and configured with appropriate credentials.
- **Docker** installed on your local machine.
- **kubectl** installed and configured.

### 2. Create CodeCommit Repositories

Create repositories for the FastAPI app, Kubernetes manifests, and CI/CD pipeline.

```sh
aws codecommit create-repository --repository-name fastapi-app-repo --repository-description "Repository for FastAPI application"
aws codecommit create-repository --repository-name k8s-manifest-repo --repository-description "Repository for Kubernetes manifests"
aws codecommit create-repository --repository-name ci-cd-repo --repository-description "Repository for CI/CD pipeline scripts"
```

### 3. FastAPI Application

Create a simple FastAPI application.

**main.py:**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
```

**Dockerfile:**

```Dockerfile
FROM tiangolo/uvicorn-gunicorn-fastapi:python3.7

COPY ./requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /app/requirements.txt

COPY ./app /app
```

**requirements.txt:**

```
fastapi
uvicorn
```

### 4. Kubernetes Deployment Manifest

**deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: <ECR_REPOSITORY_URL>:<IMAGE_TAG>
        ports:
        - containerPort: 80
```

### 5. Create ECR Repository

Create an ECR repository to store the Docker images.

```sh
aws ecr create-repository --repository-name fastapi-repo
```

### 6. Create and Configure EKS Cluster

Create an EKS cluster and configure the worker nodes.

```sh
aws eks create-cluster --name fastapi-cluster --role-arn <EKS_ROLE_ARN> --resources-vpc-config subnetIds=<SUBNET_IDS>,securityGroupIds=<SECURITY_GROUP_IDS>
aws eks wait cluster-active --name fastapi-cluster

aws eks create-nodegroup --cluster-name fastapi-cluster --nodegroup-name fastapi-node-group --node-role <NODE_ROLE_ARN> --subnets <SUBNET_IDS> --scaling-config minSize=1,maxSize=3,desiredSize=2
aws eks wait nodegroup-active --cluster-name fastapi-cluster --nodegroup-name fastapi-node-group

aws eks update-kubeconfig --name fastapi-cluster
```

### 7. Create CodeBuild Project

Create a CodeBuild project to build and push the Docker image.

**buildspec.yml:**

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin <ECR_REPOSITORY_URL>
      - REPOSITORY_URI=<ECR_REPOSITORY_URL>
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"fastapi-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

```sh
aws codebuild create-project --name fastapi-build-project --source type=CODECOMMIT,location=<CI_CD_REPO_URL> --artifacts type=NO_ARTIFACTS --environment type=LINUX_CONTAINER,computeType=BUILD_GENERAL1_SMALL,image=aws/codebuild/standard:4.0,privilegedMode=true --service-role <CODEBUILD_SERVICE_ROLE_ARN>
```

### 8. Create CodePipeline

Create a CodePipeline to automate the build and deploy process.

```sh
aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

**pipeline.json:**

```json
{
  "pipeline": {
    "name": "fastapi-pipeline",
    "roleArn": "<CODEPIPELINE_SERVICE_ROLE_ARN>",
    "artifactStore": {
      "type": "S3",
      "location": "pipeline-artifacts-bucket"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "outputArtifacts": [
              {
                "name": "source_output"
              }
            ],
            "configuration": {
              "RepositoryName": "fastapi-app-repo",
              "BranchName": "main"
            }
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "inputArtifacts": [
              {
                "name": "source_output"
              }
            ],
            "outputArtifacts": [
              {
                "name": "build_output"
              }
            ],
            "configuration": {
              "ProjectName": "fastapi-build-project"
            }
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeploy",
              "version": "1"
            },
            "inputArtifacts": [
              {
                "name": "build_output"
              }
            ],
            "configuration": {
              "ApplicationName": "fastapi-deploy",
              "DeploymentGroupName": "fastapi-deployment-group"
            }
          }
        ]
      }
    ]
  }
}
```

### 9. Deploy Kubernetes Manifests

After the build stage, update the Kubernetes manifests with the new image tag and deploy them.

**update-k8s.sh:**

```sh
#!/bin/bash

REPOSITORY_URI=$1
IMAGE_TAG=$2
K8S_MANIFEST_REPO_URL=$3

# Clone the manifest repository
git clone $K8S_MANIFEST_REPO_URL
cd k8s-manifests

# Update the deployment file
sed -i "s|<ECR_REPOSITORY_URL>:<IMAGE_TAG>|$REPOSITORY_URI:$IMAGE_TAG|g" deployment.yaml

# Commit and push the changes
git commit -am "Update image to $REPOSITORY_URI:$IMAGE_TAG"
git push origin main

# Apply the changes to the EKS cluster
kubectl apply -f deployment.yaml
```

You can add this script as a step in your CodePipeline to automatically update and deploy the Kubernetes manifests.

### Summary

1. **Set up repositories**: Use AWS CodeCommit to create repositories for your application, Kubernetes manifests, and CI/CD scripts.
2. **Create and configure ECR and EKS**: Use AWS CLI commands to create ECR for Docker images and EKS for Kubernetes.
3. **CI/CD Pipeline**: Use CodeBuild and CodePipeline to build the Docker image, push it to ECR, and update Kubernetes manifests.
4. **Deploy**: Use kubectl to deploy the updated manifests to your EKS cluster.

By following these steps and using the provided commands and scripts, you can set up a complete CI/CD pipeline for deploying a FastAPI application on AWS using AWS CLI.