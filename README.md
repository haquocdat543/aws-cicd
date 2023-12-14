# Aws-cicd-pipeline
This is a demonstration of using aws pipeline and k8s in cicd

## Prerequisites
* [git](https://git-scm.com/downloads)
* [terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
* [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [config-profile](https://docs.aws.amazon.com/cli/latest/reference/configure/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)

# Start
## 1. Clone repo
```
git clone https://github.com/haquocdat543/aws-cicd.git
cd aws-cicd
```
## 2. Modify .yaml file
### 1. pipeline.yaml
* Aws CodeCommit repo name. Line : 6,46
* Aws S3 bucket repo name. Line : 14, 50
* Region. Line : 32
* Account number. Line : 34
* Ecr repo name. Line : 36, 90
* Tag. Line : 38
## 3. Initiaize
### 1. k8s cluster
```
aws cloudformation deploy --stack-name k8s --template-file k8s.yaml
```
### 2. pipeline
```
aws cloudformation deploy --stack-name pipeline --template-file pipeline.yaml
```
## 4. Get output
```
aws cloudformation describe-stacks --query Stacks[].Outputs[*].[OutputKey,OutputValue] --output text
```
## 5. Setup and config
### 1. k8s
```
aws eks update-kubeconfig --name my-eks
```

