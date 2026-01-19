# aws_eks_app_deploy
Setting up an AWS EKS cluster and deploying a sample 2048 game application, making it accessible via an Application Load Balancer (ALB) through Kubernetes Ingress.

# 1. Prerequisites and Tools Setup
Before starting the EKS cluster creation, several command-line tools must be installed and configured on your local machine

**Installing the AWS CLI**:
Download and install the AWS CLI on your local machine. You can find installation instructions for various operating systems [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

*Configuring AWS CLI Credentials:*
Access Keys (for Programmatic Access):
- If you selected "Programmatic access" during user creation, you will receive access keys (Access Key ID and Secret Access Key).
- Store these access keys securely, as they will be used to authenticate API requests made to AWS services.

Open a terminal or command prompt and run the following command:
```
aws configure
```
Enter the access key ID and secret access key of the IAM user you created earlier.
Choose a default region and output format for AWS CLI commands.

**Installing kubectl**:
Install kubectl on your local machine. Instructions can be found [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

**Installing eksctl**:
eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installing or updating](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).


# Compute (EC2) vs Serverless Compute (fargate)
Use the --fargate flag in the command "eksctl create cluster --name cluster_name --region us-east-1 --fargate" to specify that the worker nodes (data plane) for the EKS cluster should be managed by AWS Fargate instead of traditional EC2 instances.

**Here's why to choose --fargate and what it implies**:

1. Serverless Compute: Fargate is an AWS serverless compute engine designed specifically for containers. By using --fargate, you don't have to provision, patch, or scale individual EC2 instances for your Kubernetes worker nodes. AWS handles all the underlying infrastructure management.
- Reduced Operational Overhead: This significantly reduces the operational burden on a DevOps engineer.
- You don't need to worry about: Worker node going down.
- Resource consumption on worker nodes: Setting up auto-scaling for worker nodes, Monitoring the health of individual worker nodes.
- Highly Available and Robust: The combination of a managed EKS control plane and Fargate-managed worker nodes allows you to build a highly available and stable Kubernetes cluster.
  
If you were to use "eksctl create cluster --name demo-cluster --region us-east-1" (without --fargate), you would then need to manually configure and manage your worker nodes, typically using EC2 instances, which involves more responsibility for the underlying infrastructure. If an organization has specific requirements for worker node distributions (like RHEL) or other custom specifications, then using EC2 instances might be necessary. Otherwise, Fargate is generally preferred for its simplicity and reduced management.


# 2. Create cluster using Fargate
The EKS cluster is created using the eksctl utility, which automates the provisioning of necessary AWS resources, including VPCs, public/private subnets, and IAM roles:
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
**Process**: This command initiates the creation of a managed EKS control plane and configures Fargate as the serverless compute engine for worker nodes. This process can take 10-20 minutes.
**Verification**: Once completed, the cluster can be seen in the AWS EKS console, showing details like Kubernetes version and API server endpoint. The console also provides a "Resources" tab to view cluster components like pods, daemon sets, and service accounts

# Delete the cluster
eksctl delete cluster --name demo-cluster --region us-east-1


# 3. kubeconfig Update
To interact with the newly created EKS cluster using kubectl from your local machine, the kubeconfig file needs to be updated:
```
aws eks update-kubeconfig --name demo-cluster-one --region us-east-1
```
**Purpose**: This command configures kubectl to connect to your EKS cluster, allowing you to manage Kubernetes resources directly from your terminal.

# 4. Fargate Profile Creation for Application Namespace
By default, Fargate profiles are set for default and kube-system namespaces. To deploy applications in a custom namespace using Fargate, a new Fargate profile is required:
```
eksctl create fargateprofile --cluster demo-cluster-one --name ALB-sample-app --namespace game2048
```
**Purpose**: This ensures that pods deployed within the specified `` (e.g., game2048) are scheduled on Fargate, leveraging its serverless capabilities.
**Verification**: The new Fargate profile appears in the "Compute" section of the EKS cluster overview in the AWS console.

# 5. Application Deployment (2048 Game)
The 2048 game application is deployed using a single YAML file that encompasses the namespace, deployment, service, and Ingress resources:

Command: kubectl apply -f [https://raw.githubusercontent.com/iam-veeramalla/aws-devops-zero-to-hero/main/day-22/2048-app-deploy-ingress.yaml](https://raw.githubusercontent.com/iam-veeramalla/aws-devops-zero-to-hero/main/day-22/2048-app-deploy-ingress.yaml`) (44:48)
YAML File Breakdown (45:09):
Namespace (game2048): Created first to logically separate application resources.
Deployment: Defines the 2048 game application with a specified container image and replica count (e.g., 5 replicas) (45:28).
Service: Exposes the pods internally within the cluster. It uses selectors to match pods based on labels (e.g., app.kubernetes.io/name: app2048) (46:06).
Ingress: Defines how external HTTP/HTTPS traffic is routed to the service.
Annotations: Include specific configurations for the AWS ALB Ingress Controller (e.g., kubernetes.io/ingress.class: alb, alb.ingress.kubernetes.io/scheme: internet-facing) (47:04).
Rules: Specify that traffic for the default path (/) should be forwarded to the game2048-svc service on port 80 (47:18).
Ingress Controller Role: The alb.ingress.kubernetes.io annotations instruct the ALB Ingress Controller (which would need to be deployed and configured on the cluster, though its deployment is not explicitly shown in this demo segment) to watch for this Ingress resource and provision an AWS Application Load Balancer accordingly. This ALB then routes external requests to the game2048-svc service, and subsequently to the 2048 application pods (47:25).
