Yes, it is possible to set up the entire infrastructure and CI/CD pipeline using Terraform. Below is a comprehensive guide along with the required scripts to achieve this:

### 1. Prerequisites

- **Terraform** installed on your local machine.
- **AWS CLI** configured with appropriate credentials.
- **Docker** installed on your local machine.

### 2. Directory Structure

```
fastapi-app/
├── main.py
├── Dockerfile
├── requirements.txt
k8s-manifests/
├── deployment.yaml
ci-cd/
├── main.tf
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

### 5. Terraform Script

**main.tf:**

```hcl
provider "aws" {
  region = "us-west-2"
}

resource "aws_ecr_repository" "fastapi" {
  name = "fastapi-repo"
}

resource "aws_eks_cluster" "example" {
  name     = "example-cluster"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = aws_subnet.private.*.id
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_AmazonEKSClusterPolicy,
    aws_iam_role_policy_attachment.eks_cluster_AmazonEKSServicePolicy,
  ]
}

resource "aws_iam_role" "eks_cluster" {
  name = "eks_cluster_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      },
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_iam_role_policy_attachment" "eks_cluster_AmazonEKSServicePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_iam_role" "eks_nodes" {
  name = "eks_nodes_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_nodes_AmazonEKSWorkerNodePolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "eks_nodes_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_iam_role_policy_attachment" "eks_nodes_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes.name
}

resource "aws_eks_node_group" "example" {
  cluster_name    = aws_eks_cluster.example.name
  node_group_name = "example-eks-node-group"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private.*.id

  scaling_config {
    desired_size = 2
    max_size     = 2
    min_size     = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_nodes_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.eks_nodes_AmazonEKS_CNI_Policy,
    aws_iam_role_policy_attachment.eks_nodes_AmazonEC2ContainerRegistryReadOnly,
  ]
}

resource "aws_codecommit_repository" "fastapi_app_repo" {
  repository_name = "fastapi-app-repo"
}

resource "aws_codecommit_repository" "k8s_manifest_repo" {
  repository_name = "k8s-manifest-repo"
}

resource "aws_codecommit_repository" "ci_cd_repo" {
  repository_name = "ci-cd-repo"
}

resource "aws_codebuild_project" "example" {
  name          = "example"
  service_role  = aws_iam_role.codebuild_service_role.arn
  source {
    type      = "CODECOMMIT"
    location  = aws_codecommit_repository.ci_cd_repo.clone_url_http
    buildspec = "buildspec.yml"
  }

  artifacts {
    type = "NO_ARTIFACTS"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "aws/codebuild/standard:4.0"
    type                        = "LINUX_CONTAINER"
    privileged_mode             = true
    environment_variable {
      name  = "REPOSITORY_URI"
      value = aws_ecr_repository.fastapi.repository_url
    }
  }
}

resource "aws_codepipeline" "example" {
  name = "example-pipeline"

  role_arn = aws_iam_role.codepipeline_service_role.arn

  artifact_store {
    location = aws_s3_bucket.pipeline_bucket.bucket
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeCommit"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        RepositoryName = aws_codecommit_repository.fastapi_app_repo.repository_name
        BranchName     = "main"
      }
    }
  }

  stage {
    name = "Build"

    action {
      name             = "Build"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      version          = "1"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]

      configuration = {
        ProjectName = aws_codebuild_project.example.name
      }
    }
  }

  stage {
    name = "Deploy"

    action {
      name             = "Deploy"
      category         = "Deploy"
      owner            = "AWS"
      provider         = "CodeDeploy"
      version          = "1"
      input_artifacts  = ["build_output"]

      configuration = {
        ApplicationName     = aws_codedeploy_application.example.name
        DeploymentGroupName = aws_codedeploy_deployment_group.example.name
      }
    }
  }
}

resource "aws_s3_bucket" "pipeline_bucket" {
  bucket = "pipeline-artifacts-bucket"
}

resource "aws_iam_role" "codebuild_service_role" {
  name = "codebuild_service_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codebuild.amazonaws.com"
        }
      },
    ]
  })

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "s3:GetObject",
          "s3:PutObject",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:CompleteLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:InitiateLayerUpload",
          "codecommit:GitPull",
        ],
        Effect = "Allow",
        Resource = "*",
      },
    ]
  })
}

resource "aws_iam_role" "codepipeline_service_role" {
  name = "codepipeline_service_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "codepipeline.amazonaws.com"
        }
      },
    ]
  })

  policy = jsonencode({
    Version = "2012-10-17"
    Statement =

 [
      {
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "codecommit:GitPull",
          "codedeploy:CreateDeployment",
          "codedeploy:GetApplication",
          "codedeploy:GetDeploymentGroup",
          "codedeploy:ListDeploymentGroups",
          "codedeploy:ListApplications",
        ],
        Effect = "Allow",
        Resource = "*",
      },
    ]
  })
}
```

### 6. CodeBuild Build Specification

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

### 7. Deploy and Apply Terraform

1. Initialize Terraform:
   ```sh
   terraform init
   ```

2. Plan the deployment:
   ```sh
   terraform plan
   ```

3. Apply the deployment:
   ```sh
   terraform apply
   ```

This script will set up:

- An ECR repository for the Docker images.
- An EKS cluster and node group for running the Kubernetes workloads.
- CodeCommit repositories for your application code, Kubernetes manifests, and CI/CD pipeline configurations.
- CodeBuild project to build and push Docker images.
- CodePipeline to orchestrate the CI/CD process.

### Final Notes

- Ensure the `buildspec.yml` file is in the root of your FastAPI repository.
- Replace placeholders like `<ECR_REPOSITORY_URL>` and `<IMAGE_TAG>` with actual values in your build scripts.
- You may need to adjust IAM policies and roles according to your security requirements.

This comprehensive guide outlines the steps to set up and deploy a FastAPI application using Terraform on AWS, covering CI/CD pipelines, Docker, EKS, and other necessary components.