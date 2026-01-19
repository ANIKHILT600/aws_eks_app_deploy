# aws_eks_app_deploy
Setting up an AWS EKS cluster and deploying a sample 2048 game application, making it accessible via an Application Load Balancer (ALB) through Kubernetes Ingress.

# 1. Prerequisites and Tools Setup
Before starting the EKS cluster creation, several command-line tools must be installed and configured on your local machine

**Installing the AWS CLI:**
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

**Installing kubectl:**
Install kubectl on your local machine. Instructions can be found [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

**Installing eksctl:**
eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installing or updating](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).





# Create cluster using Fargate
eksctl create cluster --name demo-cluster --region us-east-1 --fargate

# Delete the cluster
eksctl delete cluster --name demo-cluster --region us-east-1




2. EKS Cluster Creation
The EKS cluster is created using the eksctl utility, which automates the provisioning of necessary AWS resources, including VPCs, public/private subnets, and IAM roles (30:25):

Command: eksctl create cluster --name --region --fargate (31:00)
Example: eksctl create cluster --name demo-cluster-one --region us-east-1 --fargate (32:40)
Process: This command initiates the creation of a managed EKS control plane and configures Fargate as the serverless compute engine for worker nodes. This process can take 10-20 minutes (33:46).
Verification: Once completed, the cluster can be seen in the AWS EKS console (34:50), showing details like Kubernetes version and API server endpoint (36:33). The console also provides a "Resources" tab to view cluster components like pods, daemon sets, and service accounts (35:34).
3. kubeconfig Update
To interact with the newly created EKS cluster using kubectl from your local machine, the kubeconfig file needs to be updated (40:41):

Command: aws eks update-kubeconfig --name --region (41:07)
Example: aws eks update-kubeconfig --name demo-cluster-one --region us-east-1 (41:18)
Purpose: This command configures kubectl to connect to your EKS cluster, allowing you to manage Kubernetes resources directly from your terminal.
4. Fargate Profile Creation for Application Namespace
By default, Fargate profiles are set for default and kube-system namespaces. To deploy applications in a custom namespace using Fargate, a new Fargate profile is required (39:40):

Command: eksctl create fargateprofile --cluster --name --namespace (42:04)
Example: eksctl create fargateprofile --cluster demo-cluster-one --name ALB-sample-app --namespace game2048 (42:40)
Purpose: This ensures that pods deployed within the specified `` (e.g., game2048) are scheduled on Fargate, leveraging its serverless capabilities.
Verification: The new Fargate profile appears in the "Compute" section of the EKS cluster overview in the AWS console (43:13).
5. Application Deployment (2048 Game)
The 2048 game application is deployed using a single YAML file that encompasses the namespace, deployment, service, and Ingress resources (44:20):

Command: kubectl apply -f [https://raw.githubusercontent.com/iam-veeramalla/aws-devops-zero-to-hero/main/day-22/2048-app-deploy-ingress.yaml](https://raw.githubusercontent.com/iam-veeramalla/aws-devops-zero-to-hero/main/day-22/2048-app-deploy-ingress.yaml`) (44:48)
YAML File Breakdown (45:09):
Namespace (game2048): Created first to logically separate application resources.
Deployment: Defines the 2048 game application with a specified container image and replica count (e.g., 5 replicas) (45:28).
Service: Exposes the pods internally within the cluster. It uses selectors to match pods based on labels (e.g., app.kubernetes.io/name: app2048) (46:06).
Ingress: Defines how external HTTP/HTTPS traffic is routed to the service.
Annotations: Include specific configurations for the AWS ALB Ingress Controller (e.g., kubernetes.io/ingress.class: alb, alb.ingress.kubernetes.io/scheme: internet-facing) (47:04).
Rules: Specify that traffic for the default path (/) should be forwarded to the game2048-svc service on port 80 (47:18).
Ingress Controller Role: The alb.ingress.kubernetes.io annotations instruct the ALB Ingress Controller (which would need to be deployed and configured on the cluster, though its deployment is not explicitly shown in this demo segment) to watch for this Ingress resource and provision an AWS Application Load Balancer accordingly. This ALB then routes external requests to the game2048-svc service, and subsequently to the 2048 application pods (47:25).
