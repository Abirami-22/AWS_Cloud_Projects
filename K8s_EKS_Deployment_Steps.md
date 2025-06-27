<h1>Amazon EKS</h1>

Elastic Kubernetes Service is a managed Kubernetes service provided by AWS. It handles the Kubernetes control plane (also known as the master nodes) for you.
EKS can be considered the managed Kubernetes control plane / master node.

<h3>✅ What EKS Manages (Control Plane): </h3>
When you create an EKS cluster, AWS provisions and manages the following control plane components:

API Server

etcd (key-value store)

Controller Manager

Scheduler

Cloud Integration for IAM, VPC, etc.

You don’t manage or access the control plane servers directly — AWS handles high availability, security patches, and scaling of these components.

<h3>🔸 What You Manage (Data Plane):</h3>

You (the user) manage:

Worker nodes (EC2, Fargate, or other compute backends)

Kubernetes workloads (pods, deployments, services, etc.)

Networking and storage configurations
