# Brain Tasks App - AWS EKS Deployment

A React app deployed with Docker, ECR, EKS, CodeBuild and CodePipeline.

- **GitHub:** https://github.com/abirami-0901/brain-tasks-app
- **Region:** ap-south-1 (Mumbai)
- **Load Balancer ARN:**
  `arn:aws:elasticloadbalancing:ap-south-1:827670477976:loadbalancer/net/a2e85e68834d548c4957555b66fe88ba/052cc706798d6488`
- **App URL:** http://a2e85e68834d548c4957555b66fe88ba-052cc706798d6488.elb.ap-south-1.amazonaws.com

## How it works

```
GitHub push -> CodePipeline -> CodeBuild -> ECR (image) -> EKS -> Load Balancer
```

1. **Source:** CodePipeline watches the `main` branch on GitHub.
2. **Build:** CodeBuild runs `buildspec.yml`. It builds the Docker image, pushes it to ECR, then runs `kubectl apply` to deploy to EKS.
3. **Run:** EKS runs 2 pods. A Network Load Balancer sends traffic from port 80 to the app on port 3000.
4. **Logs:** Build and deploy logs go to CloudWatch (`/aws/codebuild/brain-tasks-build`).

## Files

| File | Purpose |
|---|---|
| `Dockerfile`, `nginx.conf` | Serve the app with nginx on port 3000 |
| `buildspec.yml` | CodeBuild steps: build, push, deploy |
| `k8s/deployment.yaml` | 2 replicas of the app |
| `k8s/service.yaml` | LoadBalancer service (NLB) |

## Setup steps

**1. Build and test locally**
```bash
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
docker build -t brain-tasks-app .
docker run -d -p 3000:3000 brain-tasks-app
```
Open http://localhost:3000

![Docker build](screenshots/01-docker-build.png)
![Localhost](screenshots/02-docker-run-localhost.png)

**2. Push the image to ECR**
```bash
aws ecr create-repository --repository-name brain-tasks-app --region ap-south-1
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REPO_URI=$AWS_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/brain-tasks-app
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
docker tag brain-tasks-app:latest $REPO_URI:latest
docker push $REPO_URI:latest
```

![ECR repo](screenshots/03-ecr-repository.png)
![Image pushed](screenshots/04-ecr-image-pushed.png)

**3. Create the EKS cluster**
```bash
eksctl create cluster --name brain-tasks-cluster --region ap-south-1 \
  --nodegroup-name brain-tasks-nodes --node-type t3.medium --nodes 2 --managed
kubectl get nodes
```

![Cluster active](screenshots/05-eks-cluster-active.png)
![Nodes](screenshots/06-kubectl-nodes.png)

**4. Deploy to Kubernetes**
```bash
kubectl apply -f k8s/
kubectl get pods,svc
```

![Pods and service](screenshots/07-kubectl-pods-services.png)
![App on load balancer](screenshots/08-app-loadbalancer-url.png)

**5. Create the CodeBuild project** (`brain-tasks-build`)
- Source: GitHub repo, branch `main`
- Environment: Amazon Linux, Standard, **Privileged mode ON**
- Buildspec: `buildspec.yml`
- Role permissions: ECR access, `eks:DescribeCluster`, and the role added to the EKS cluster with `eksctl create iamidentitymapping`

![CodeBuild project](screenshots/09-codebuild-project.png)
![Build success](screenshots/10-codebuild-success.png)

**6. Create the CodePipeline** (`brain-tasks-pipeline`)
- Source: GitHub (via GitHub App), branch `main`
- Build: CodeBuild project `brain-tasks-build`
- The deploy to EKS runs in the build stage (`kubectl apply` in `buildspec.yml`)

![Pipeline success](screenshots/11-codepipeline-success.png)

## Monitoring

- Build and deploy logs: CloudWatch Logs, group `/aws/codebuild/brain-tasks-build`
- App logs: `kubectl logs deployment/brain-tasks-app`

![Log groups](screenshots/12-cloudwatch-log-groups.png)
![Build logs](screenshots/13-cloudwatch-build-logs.png)
![App logs](screenshots/13b-app-logs-kubectl.png)

## Load Balancer

![Load balancer ARN](screenshots/14-loadbalancer-arn.png)

No dependencies are required

No build process is required

No package.json is needed

