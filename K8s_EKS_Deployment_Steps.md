<h1>Amazon EKS</h1>

Elastic Kubernetes Service is a managed Kubernetes service provided by AWS. It handles the Kubernetes control plane (also known as the master nodes) for you.
EKS can be considered the managed Kubernetes control plane / master node.

<h3>✅ What EKS Manages (Control Plane): </h3>
When you create an EKS cluster, AWS provisions and manages the following control plane components:

1. API Server

2. etcd (key-value store)

3. Controller Manager

4. Scheduler

5. Cloud Integration for IAM, VPC, etc.

You don’t manage or access the control plane servers directly — AWS handles high availability, security patches, and scaling of these components.

<h3>🔸 What You Manage (Data Plane):</h3>

You (the user) manage:

1. Worker nodes (EC2, Fargate, or other compute backends)

2. Kubernetes workloads (pods, deployments, services, etc.)

3. Networking and storage configurations

<h1>Prerequisites</h1>
Before we begin, ensure you have the following prerequisites in place:

**AWS Account:** You will be deploying resources to AWS, so you need an AWS account with sufficient permissions (Administrator or a role that can create EKS clusters, IAM roles, VPCs, etc.).

**AWS CLI:** Installed and configured with your AWS credentials. This lets you interact with AWS from the command line (e.g., to fetch cluster details or configure credentials).

**kubectl:** The Kubernetes CLI for interacting with the cluster. Ensure you have kubectl installed (matching your cluster’s Kubernetes version) and on your PATH. You can install it via Amazon’s instructions (for example, by downloading the binary from an S3 URL and making it executable).

**eksctl:** A CLI tool to create and manage EKS clusters easily. We'll use eksctl to create the Kubernetes cluster and to set up some addons. Install eksctl v**≥0.148.0** (latest version) on your system.

**Helm:** The package manager for Kubernetes, used to deploy the application’s Helm chart. Install Helm v3 on your system.

**Docker (optional):** If you want to test locally or build images, Docker is useful. (For this deployment, you won’t need to build images manually because pre-built images are available on Docker Hub.)

<h1>EKS Cluster Setup</h1>
Now we will set up an EKS Kubernetes cluster to host the application. This includes creating the cluster itself and then configuring necessary add-ons such as IAM OIDC provider, the AWS Load Balancer Controller (for ingress), and the EBS CSI driver (for persistent storage). The following steps assume you’re operating in the us-east-1 region (North Virginia); you can change the region and names as needed.

```apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: demo-cluster-three-tier-1         # Cluster name
  region: us-east-1                       # AWS region
  version: "1.29"                         # Kubernetes version (adjust as needed)

vpc:
  cidr: "10.0.0.0/16"                     # Custom VPC CIDR
  nat:
    gateway: Single                       # Use a single NAT Gateway
  subnets:
    public:
      us-east-1a: { cidr: "10.0.1.0/24" }
      us-east-1b: { cidr: "10.0.2.0/24" }
    private:
      us-east-1a: { cidr: "10.0.3.0/24" }
      us-east-1b: { cidr: "10.0.4.0/24" }

nodeGroups:
  - name: general-workers                 # Node group name
    instanceType: t3.medium               # EC2 instance type
    desiredCapacity: 2                   # Start with 2 EC2 nodes
    minSize: 2
    maxSize: 4
    volumeSize: 20                       # EBS volume size (in GB)
    ssh:
      allow: true
      publicKeyName: my-ec2-keypair      # Replace with your EC2 key pair name
    iam:
      withAddonPolicies:
        autoScaler: true                 # Optional: for cluster auto-scaler
        ebs: true                        # Optional: if using EBS volumes
    tags:
      Name: demo-node                    # Will be applied as EC2 Name tag
      environment: dev

```
Amazon EKS (Elastic Kubernetes Service) can be set up using either:

<h6>🖥️ 1. EC2-based Worker Nodes</h6>
These are regular EC2 instances that serve as Kubernetes worker nodes.

Use case: When you need more control over the compute environment (e.g., GPU instances, custom AMIs, SSH access, etc.).

Setup tools: eksctl, CloudFormation, or console.

Cost: You pay for EC2 instances separately (instance hours + storage).

<h6>☁️ 2. AWS Fargate (Serverless)</h6>
Fargate runs your pods without provisioning or managing EC2 nodes.

Use case: When you want fully managed compute, scale automatically, and don't want to manage EC2 instances.

No need to patch or manage worker nodes.

Cost: You pay per vCPU and memory used by your pods.

<h6>🔁 Hybrid Option (EC2 + Fargate)</h6>
You can even mix both EC2 and Fargate profiles in the same cluster:

Run critical workloads on EC2 for performance/customization.

Use Fargate for bursty or small workloads that need minimal ops.


