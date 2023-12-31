# Aws-cicd-pipeline
This is a demonstration of using aws pipeline and k8s in cicd

## Prerequisites
* [git](https://git-scm.com/downloads)
* [terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
* [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [config-profile](https://docs.aws.amazon.com/cli/latest/reference/configure/)
* [aws-codecommit-account](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [kubectl-argo-rollouts](https://github.com/argoproj/argo-rollouts/blob/master/docs/installation.md)

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
### 2. pipeline
#### 1. Clone aws codecommit repo
Copy https url from Output
```
git clone <your-aws-codecommit-https-url>
```
Then enter your aws code commit username and password. [aws-codecommit-account](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html)
```
cd <your-aws-codecommit-repo-name>
```
then create a frontend project inside it. In my case it is `vue`
Add `Dockerfile` . In my case it is [vue-dockerization](https://v2.vuejs.org/v2/cookbook/dockerize-vuejs-app)
Add `buildspec.yaml` at `root` folder
Create a commit and push it to `aws-codecommit-repo`

## 6. Start build docker image
Get `codebuild-project-name` and `Ecr-repo-name` from `Output`:
```
aws codebuild start-build --project-name <your-codebuild-project-name>
```
Wait a little bit and check image :
```
aws ecr describe-images --repository-name <your-ecr-repo-name>
```
## 7. Argo Rollouts
### 1. Install kubectl argo rollouts
Find compatible version and platform here. [kubectl-argo-rollouts](https://github.com/argoproj/argo-rollouts/blob/master/docs/installation.md)
```
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
kubectl argo rollouts version
```
### 2. Install argocd rollouts CRD
```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```
### 3. Login to ECR 

You need to replace variables before execute following commands :
* $AWS_DEFAULT_REGION
* $AWS_ACCOUNT_ID
```
aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
```
Or you can go to `AWS` > `ECR` > `Private Registry` > `Repositories` > `Your Repo` > `View Push Commands` > copy and execute first command

### 4. Configure Secret
#### 1. generated file
```
cat ~/.docker/config.json | base64
```
Copy the output and paste to `secret.yaml` file :
#### 2. create docker login secret from config.json file
```
kubectl create secret generic my-registry-key \
--from-file=.dockerconfigjson=.docker/config.json \
--type=kubernetes.io/dockerconfigjson

kubectl create secret generic my-registry-key --from-file=.dockerconfigjson=.docker/config.json --type=kubernetes.io/dockerconfigjson

kubect get secret
```
#### 3. create docker login secret with login credentials
```
kubectl create secret docker-registry my-registry-key \
--docker-server=https://private-repo \
--docker-username=user \
--docker-password=pwd

kubectl create secret docker-registry my-registry-key --docker-server=https://private-repo --docker-username=user --docker-password=pwd
```
### 5. Edit bluegreen-rollout.yaml
Modify your imagePullSecret to `Your registry secret`
### 6. Apply bluegreen
```
kubectl apply -f bluegreen-rollout.yaml
```


### 7. Patch services
```
kubectl patch svc bluegreen-demo -n default -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc bluegreen-demo-preview -n default -p '{"spec": {"type": "LoadBalancer"}}'
```
### 8. Access services
```
kubectl get svc
```
Copy both `Loadbalancer-dns` and open it in your browser
### 9. Build new image 
* Update frontend app to version 2 
* Update image version in `buildspec.yml`
* Start new build
### 10. Update bluegreen image
Change image version tag in bluegreen.yml
```
kubectl apply -f bluegreen-rollout.yaml
```
Copy both `LoadbalancerDNS` and open it in browser
### 11. Promote green
```
kubectl argo rollouts promote bluegreen-demo
```

