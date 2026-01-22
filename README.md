# AWS-EKS-APP-DEPLOY
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
The EKS cluster is created using the eksctl utility, which automates the provisioning of necessary AWS resources, including VPCs, public/private subnets, IAM roles and the EKS control plane:
```
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
```
eksctl get cluster --region us-east-1
```
**Process**: This command initiates the creation of a managed EKS control plane and configures Fargate as the serverless compute engine for worker nodes. This process can take 10-20 minutes.

**Verification**: Once completed, the cluster can be seen in the AWS EKS console, showing details like Kubernetes version and API server endpoint. The console also provides a "Resources" tab to view cluster components like pods, daemon sets, and service accounts.


# 3. kubeconfig Update
To interact with the newly created EKS cluster using kubectl from your local machine, the kubeconfig file needs to be updated:
```
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```
**Purpose**: This command configures kubectl to connect to your EKS cluster, allowing you to manage Kubernetes resources directly from your terminal.

Note: You can set AWS CLI region as:
```
aws configure set region us-east-1
```

# 4. Fargate Profile Creation for Application Namespace
By default, Fargate profiles are set for default and kube-system namespaces. You can use default profile. But if you want to deploy applications in a custom namespace (like example: game2048) using Fargate, a new Fargate profile (like example: alb-sample-app) is required:
```
eksctl create fargateprofile --cluster demo-cluster --name alb-sample-app --namespace game-2048
```

**Purpose**: This ensures that pods deployed within the specified namespace (e.g., game2048) are scheduled on Fargate, leveraging its serverless capabilities.

**Verification**: The new Fargate profile appears in the "Compute" section of the EKS cluster overview in the AWS console.

# 5. Deploy 2048 Game Application, Service, and Ingress
The 2048 game application is deployed using a single YAML file that encompasses the namespace, deployment, service, and Ingress resources. Refering from [kubernetes-sigs](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/examples/2048/2048_full.yaml):
```
kubectl apply -f https://raw.githubusercontent.com/ANIKHILT600/aws_eks_app_deploy/refs/heads/main/2048-app.yaml
```
**YAML File Breakdown**:
- Namespace (game2048): Created first to logically separate application resources.
- Deployment: Defines the 2048 game application with a specified container image and replica count (e.g., 5 replicas).
- Service: Exposes the pods internally within the cluster. It uses selectors to match pods based on labels (e.g., app.kubernetes.io/name: app2048).
- Ingress: Defines how external HTTP/HTTPS traffic is routed to the service.
- Annotations: Include specific configurations for the AWS ALB Ingress Controller (e.g., kubernetes.io/ingress.class: alb, alb.ingress.kubernetes.io/scheme: internet-facing).
- Rules: Specify that traffic for the default path (/) should be forwarded to the game2048-svc service on port 80.

**Check deployment**
1. To check the pods
```
kubectl get pods -n game-2048
```
2. To check the service
```
kubectl get svc -n game-2048
```
3. To check the ingress
```
kubectl get ingress -n game-2048
```

**Understanding Ingress and Ingress-controller**:

**Ingress**

*What it is*: Ingress is a Kubernetes resource.

Purpose: Its primary purpose is to allow external customers or users to access applications deployed inside the Kubernetes cluster (19:01-19:08). It essentially routes traffic from outside to services within the cluster.

*How it works*:
A DevOps engineer writes an ingress.yaml file.
This file specifies rules for routing. For example, it defines that if a user accesses a specific domain and path (e.g., example.com/ABC), the request should be forwarded to a particular Kubernetes service.
The service then directs the traffic to the appropriate pod where the application is running.

*Benefit over LoadBalancer Service*: 
While the LoadBalancer service type can also expose applications publicly, it becomes very costly if you have many applications, as each would require its own load balancer. Ingress provides a more cost-effective solution by allowing a single load balancer (managed by the Ingress Controller) to handle routing for multiple applications.

**Ingress Controller**:

*What it is*: The Ingress Controller is a component within Kubernetes that is responsible for fulfilling the Ingress rules. It's essentially the actual "traffic cop" that implements the routing defined in the Ingress resources.

*How it works*:
Ingress Controllers are typically supported by various load balancers and platforms, such as Nginx, F5, or in the video's context, AWS ALB (Application Load Balancer).
They can be deployed into the Kubernetes cluster using Helm charts or plain YAML manifests.
The Ingress Controller continuously watches for Ingress resources that are created in the cluster.
When an Ingress Controller finds an Ingress resource (specifically, one that matches its ingressClassName, like alb for the ALB Ingress Controller), it takes action.
It then configures this load balancer with the rules specified in the Ingress resource, ensuring that external user requests are correctly routed to the application services and pods within the cluster.

**In summary**, the Ingress resource defines how external traffic should be routed, and the Ingress Controller is the active component that makes that routing happen by provisioning and configuring the necessary load balancers and networking infrastructure 

# 6. Setting up Ingress-Controller

**Why IAM OIDC Integration is Needed for EKS Clusters**:

*Problem*: Kubernetes applications (pods) running within your EKS cluster often need to interact with other AWS services, such as S3 buckets, EKS control plane, CloudWatch or any other AWS service.
*The Challenge (without IAM OIDC)*: How do you grant these Kubernetes pods the necessary permissions to securely access AWS services? Traditionally, AWS resources (like EC2 instances) use IAM roles to get permissions. Simply embedding AWS credentials directly into pods is insecure and unmanageable.

*The Solution: IAM Roles for Service Accounts (IRSA) via OIDC*:
IAM OIDC integration allows your EKS cluster to act as an identity provider for AWS IAM. This setup enables you to attach specific IAM roles directly to Kubernetes Service Accounts within your cluster. When a pod runs and uses a service account with an attached IAM role, the pod can assume that IAM role and inherit its permissions to access AWS services.

*Benefit*: This provides a secure and granular way for your Kubernetes applications to authenticate and authorize themselves with AWS services, without hardcoding credentials or using less secure methods.

**Configure IAM OIDC provider**:
Commands to configure IAM OIDC provider is
```
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

**Download IAM policy**
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```
Note: If above command fails due to curl, use below power shell commad:
```
Invoke-WebRequest -Uri https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json -OutFile iam_policy.json
```

**Create IAM Policy**
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

**Create IAM Role**
```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
Note: Make sure to replace clusterName & your-aws-account-id.

**Deploy Ingress-Controller (alb)**

*Add helm repo*:

Helm is used to streamline the deployment of the ALB Ingress Controller, ensuring it's set up correctly to manage incoming traffic for the 2048 game application.
```
helm repo add eks https://aws.github.io/eks-charts
```

*Update the repo*:
```
helm repo update eks
```

*Install helm*:
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<your-region> \
  --set vpcId=<your-vpc-id>
```
Note: Make sure to replace clusterName, region & vpcId.


*Verify that the deployments are running*:
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

# What happens after the Ingress controller is configured and deployed:
- *Ingress Resource Created*: You define Ingress rules (hostnames, paths, backend services) in a YAML file and apply it to Kubernetes.

- *ALB Ingress Controller Detects*: The deployed ALB Ingress Controller watches for this new Ingress resource .

- *AWS ALB Provisioned*: The controller automatically provisions an AWS Application Load Balancer (ALB) based on your Ingress rules and annotations.

- *Ingress ADDRESS Updated*: The Kubernetes Ingress resource's ADDRESS field is then populated with the DNS name of this newly created ALB.

- *Traffic Routing*: External traffic flows to the ALB's DNS name, which then routes it to your application pods within the EKS cluster 


**You can either get the ALB dns from kubctl get ingress -n game-2048 or go to AWS console --> ec2 --> load balancer, and try it on browser.**


**Delete the cluster**
```
eksctl delete cluster --name demo-cluster --region us-east-1
```


# Troubleshooting if facing issue while deploying ingress-controller
Here are the commands corresponding to the troubleshooting steps:

1. Check Pod Status: To see if your Ingress controller pods are running, replace `` with the namespace where your Ingress controller is deployed (commonly kube-system or ingress-nginx).

kubectl get pods -n kube-system

2. Check Pod Logs: If a pod is not running or you suspect an issue, check its logs. Replace `` with the exact name of one of your Ingress controller pods.

kubectl logs -n kube-system

You can also add -f to follow the logs in real-time.

kubectl logs -f -n kube-system

3. Check Controller Service: To inspect the Service associated with your Ingress controller and find its external IP or hostname.

kubectl get svc -n game-2048

4. Examine the Ingress Resource: Get detailed information about your specific Ingress resource. Replace `` with the name of your Ingress.

kubectl describe ingress  -n game-2048

5. List all Ingresses: This helps you see if your Ingress was created successfully and its current status.

kubectl get ingress --all-namespaces

6. Check Service Definition: Verify the Kubernetes Service that your Ingress rule points to. Replace and.

kubectl describe svc  -n game-2048

7. Edit deployment and check if any error
   
kubectl edit deployment -n kube-system aws-load-balancer-controller







