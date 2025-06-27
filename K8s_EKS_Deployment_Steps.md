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
