# Deploy-a-Node.js-Todo-App-to-AWS-Docker-ECR-ECS-Fargate-# Deploy a Node.js Todo App on AWS (Docker → ECR → ECS Fargate)

A hands-on DevOps project: containerize a Node.js todo app with Docker, push the image to Amazon ECR, and run it on AWS ECS Fargate.

## Architecture

```
Source code → Docker image → Amazon ECR → ECS Fargate task → Browser (port 8000)
```

## Tech stack

- Node.js (Express)
- Docker
- AWS: EC2 (build server), ECR, ECS Fargate, IAM, VPC / Security Groups

## Prerequisites

- AWS account and an IAM user with the `AmazonEC2ContainerRegistryPowerUser` policy
- An Ubuntu EC2 instance (t3.micro is enough) for building the image
- Docker and AWS CLI v2 installed on that instance

## Configuration used

| Item | Value |
|---|---|
| Region | ap-southeast-2 |
| ECR repository | ecr-demo |
| ECS cluster | ecr-demo-cluster |
| Launch type | Fargate (2 vCPU / 8 GiB) |
| Container ports | 80, 8000 |

## Step-by-step

### 1. Install Docker and allow your user to run it
```bash
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

### 2. Install AWS CLI v2
```bash
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 3. Configure credentials
```bash
aws configure
aws configure set region ap-southeast-2
aws sts get-caller-identity
```
> Never commit or share access keys. Prefer an IAM role attached to the EC2 instance.

### 4. Create the ECR repository
```bash
aws ecr create-repository --repository-name ecr-demo --region ap-southeast-2
```

### 5. Log in to ECR
```bash
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com
```

### 6. Build, tag and push the image
```bash
docker build -t ecr-demo .
docker tag ecr-demo:latest <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/ecr-demo:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-southeast-2.amazonaws.com/ecr-demo:latest
```

### 7. Run on ECS Fargate
1. ECS → Clusters → create `ecr-demo-cluster`.
2. Task definitions → create a Fargate task definition using the ECR image URI, with container ports 80 and 8000.
3. Run the task in the cluster with **Public IP enabled**.

### 8. Open the port
Add an inbound rule to the task's security group:

| Type | Port | Source |
|---|---|---|
| Custom TCP | 8000 | My IP (or 0.0.0.0/0 for a quick test) |

### 9. Test
```
http://<task-public-ip>:8000/todo
```
You can check from inside the VPC first:
```bash
curl -v http://<task-private-ip>:8000     # expect: 302 → /todo
```

## Troubleshooting (issues I hit and fixed)

| Problem | Cause | Fix |
|---|---|---|
| `permission denied ... docker.sock` | New docker group not active in the shell | `newgrp docker` or log out and back in |
| `NoCredentials` from AWS CLI | No credentials on the server; `aws login` needs a browser | Use `aws configure` or attach an IAM role to the EC2 |
| `AccessDenied ... ecr:GetAuthorizationToken` | IAM user had no ECR permissions | Attach `AmazonEC2ContainerRegistryPowerUser` |
| `docker build requires 1 argument` | Missing `.` at the end | `docker build -t ecr-demo .` |
| Browser `ERR_TIMED_OUT` | Security group did not allow the port | Add inbound rule for TCP 8000 |
| "You may not specify an IPv4 CIDR for an existing referenced group id rule" | Edited an existing rule instead of adding one | Cancel, then click **Add rule** for a new row |
| Still timing out after saving rules | One rule had port 0, another had the wrong source | Delete them and add Custom TCP 8000 from the correct source |

**Debugging tips**
- Timeout means a firewall is blocking. Connection refused means nothing is listening.
- Test from inside the VPC first to separate app problems from network problems.
- Always check the port and source columns after saving security group rules.
- Resources only appear in the region where you created them.

## Security notes

- Do not commit access keys, `.env` files or `~/.aws/credentials`.
- Replace `0.0.0.0/0` with your own IP, or use a load balancer.
- Use a dedicated security group, not the default one.
- Stop the Fargate task and EC2 instance when finished to avoid charges.

## Cleanup

```bash
aws ecs list-tasks --cluster ecr-demo-cluster --region ap-southeast-2
aws ecs stop-task --cluster ecr-demo-cluster --task <task-id> --region ap-southeast-2
aws ecr delete-repository --repository-name ecr-demo --force --region ap-southeast-2
```

## Next improvements

- ECS service behind an Application Load Balancer for a stable URL
- Private subnets with VPC endpoints for ECR, S3 and CloudWatch Logs
- CI/CD with Jenkins or GitHub Actions
- Infrastructure as Code with Terraform
- CloudWatch logging with the `awslogs` driver

## What I learned

Docker image workflow, AWS CLI and IAM permissions, ECR and ECS Fargate, security group rules, and layered troubleshooting (app → container → network → firewall).
