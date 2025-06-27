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

| Feature          | EC2 Worker Nodes         | Fargate                     |
| ---------------- | ------------------------ | --------------------------- |
| Control          | Full (custom AMIs, SSH)  | Minimal (fully managed)     |
| Scaling          | Manual / Auto Scaling    | Automatic                   |
| Pricing          | Per EC2 instance         | Per vCPU + memory used      |
| Setup Complexity | Higher                   | Lower                       |
| Use Case         | Complex, heavy workloads | Simple, on-demand workloads |


<h3>🖥️ 1. EC2-based Worker Nodes</h3>

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

<h3>2. Fargate Based EKS cluster</h3>
<h3>✅ Prerequisites</h3>

eksctl installed – Install Guide

kubectl installed

AWS CLI configured with appropriate permissions

IAM role with EKS permissions

Region selected (e.g., us-east-1)

🛠️ Step-by-Step: Create EKS Cluster with Fargate
<h3>✅ 1. Create a simple config file for eksctl (e.g. fargate-cluster.yaml)</h3>

```apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fargate-demo-cluster
  region: us-east-1

fargateProfiles:
  - name: fp-default
    selectors:
      - namespace: default
      - namespace: kube-system
```

Once done, eksctl will update your kubeconfig (usually ~/.kube/config) so that kubectl is pointed at the new cluster. You can verify the cluster by running a simple command, for example:

```kubectl get nodes```

to see if the nodes are up (this requires kubectl configured and in the same environment where eksctl ran).

<h1>2. Enable IAM OIDC Provider for the Cluster</h1>

Amazon EKS uses an OpenID Connect (OIDC) provider to allow Kubernetes service accounts to assume IAM roles. This is needed for certain cluster add-ons (like the ALB controller and CSI driver) to interact with AWS resources securely using IAM. We will ensure the IAM OIDC provider is associated with our cluster.

**Check and Associate OIDC:** Use the AWS CLI and eksctl to set up the provider if not already present:

```export cluster_name="demo-cluster-three-tier-1"
# Get the OIDC issuer ID for the cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f5)
# Check if OIDC provider is already associated
aws iam list-open-id-connect-providers | grep "$oidc_id" || echo "OIDC provider not found"
```

If the above check returns nothing (no OIDC provider found), associate one with the cluster by running:

```eksctl utils associate-iam-oidc-provider --cluster "$cluster_name" --approve```

This command configures the IAM OIDC provider for your EKS cluster. With OIDC in place, Kubernetes service accounts can later be linked to IAM roles via IAM service account annotations – a requirement for the AWS Load Balancer Controller and EBS CSI driver.

<h1>3. Install the AWS Load Balancer Controller (ALB Ingress Controller)</h1>

**Why we need ALB Ingress Controller**

The AWS ALB Ingress Controller (now called AWS Load Balancer Controller) is used in Kubernetes on AWS to automatically provision and manage Application Load Balancers (ALBs) in response to Kubernetes Ingress or Service resources.

To expose the microservices to the internet, we’ll use an Application Load Balancer (ALB) Ingress Controller for AWS EKS. This controller will watch Ingress resources in the cluster and create/manage an AWS ALB accordingly, allowing external traffic to reach our services. The ALB controller needs specific IAM permissions and will run as a deployment in our cluster. We’ll set it up step by step:

a. Create IAM Policy for ALB Controller: AWS provides a pre-defined IAM policy JSON for the ALB controller. Download this policy document and create an IAM policy from it:

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

The first command fetches the IAM permissions policy for the AWS Load Balancer Controller, and the second command creates a new IAM policy called AWSLoadBalancerControllerIAMPolicy in your AWS account using that document.

b. Create an IAM Role for the ALB Controller: Next, we create an IAM role and Kubernetes service account for the ALB controller to use. We can do this in one step with eksctl, which will create an IAM role and associate it with a K8s service account named aws-load-balancer-controller in the kube-system namespace:

```
eksctl create iamserviceaccount --cluster=$cluster_name --namespace=kube-system --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
Replace <YOUR_AWS_ACCOUNT_ID> with your AWS account number. This command creates an IAM role AmazonEKSLoadBalancerControllerRole with the policy we made, and links it to the aws-load-balancer-controller service account in the cluster. The --approve flag automatically applies the changes. Essentially, this grants the ALB Ingress Controller pod the necessary AWS permissions (via IAM) to create and manage load balancers, security groups, etc., on our behalf.

c. Deploy the ALB Controller via Helm: Now we deploy the controller into the cluster using Helm. First, add the AWS EKS Helm charts repository and update it, so we have access to the ALB controller chart:

```

helm repo add eks https://aws.github.io/eks-charts
helm repo update

```
Next, install the AWS Load Balancer Controller chart from that repo:

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=$cluster_name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR_VPC_ID>
```

In the above command, make sure to replace <YOUR_VPC_ID> with the ID of the VPC where your EKS cluster is running (if you used eksctl without special configuration, you can find the VPC ID via aws eks describe-cluster --name $cluster_name --query "cluster.resourcesVpcConfig.vpcId"). We also specify the cluster’s name and region. We set serviceAccount.create=false and serviceAccount.name=aws-load-balancer-controller because we have already created the service account with an IAM role in the previous step. The Helm chart will then use that existing service account.

This Helm installation will create a deployment called aws-load-balancer-controller in the kube-system namespace. After a minute, verify that the controller pod is running:

```
kubectl get deployment -n kube-system aws-load-balancer-controller

```
You should see the deployment with at least 1/1 pod available, indicating the ALB Ingress Controller is up. At this point, the cluster is ready to create AWS Load Balancers whenever we define an Ingress resource.

<h1>4. Enable the EBS CSI Driver (Persistent Storage)</h1>
Our application includes stateful services (MongoDB, MySQL, Redis), which require persistent storage in Kubernetes. AWS EKS supports the EBS CSI Driver to provision Amazon EBS volumes for Kubernetes PersistentVolumeClaims. We need to install this driver (as an EKS add-on) so that our database pods can have durable storage. Before installing, we must create an IAM role for the CSI driver.

a. Create IAM Role for EBS CSI: We will create an IAM role that grants permissions for EBS volume operations, and associate it with the EBS CSI driver’s service account:

```
eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster $cluster_name \
  --role-name "AmazonEKS_EBS_CSI_DriverRole" --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```
This command creates an IAM role named AmazonEKS_EBS_CSI_DriverRole with the Amazon-managed policy AmazonEBSCSIDriverPolicy attached (which allows Kubernetes to manage EBS volumes). We used --role-only because EKS Add-ons will create the service account for us; we just need the role ready.

b. Install the EBS CSI Driver Add-on: Now enable the EBS CSI driver for your cluster:

```
eksctl create addon --name aws-ebs-csi-driver --cluster $cluster_name \
  --service-account-role-arn arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```
This command tells EKS to install the official aws-ebs-csi-driver add-on in the cluster, using the IAM role we just created for its service account. After a short time, the EBS CSI driver pods will be running in kube-system namespace (you can check with kubectl get pods -n kube-system -l app=ebs-csi-controller). Once this is complete, your cluster can dynamically provision EBS volumes for any PersistentVolumeClaims that our application’s StatefulSets might create. In summary, the cluster is now configured with everything needed: an OIDC provider, the ALB ingress controller, and the EBS CSI driver – fully ready to host our microservices application.

<h1>5.Deploying the Microservices with Helm</h1>

With the infrastructure in place, we can deploy the Robot Shop application onto the EKS cluster using Helm. The GitHub repository we cloned contains a pre-built Helm chart that defines all the Kubernetes objects (Deployments, Services, StatefulSets, etc.) for the 3-tier app. We will use that chart to install the whole stack in one go.

**1. Clone the Repository (if not already done)**

If you haven’t already, clone the project repository and navigate to the Helm chart directory:

```
git clone https://github.com/AbhishekTechie10/three-tier-architecture-demo.git
cd three-tier-architecture-demo/EKS/helm
```

This repository contains the code and manifests for the Robot Shop application. In particular, the EKS/helm directory contains a Chart.yaml, templates, and an ingress.yaml file – everything needed to deploy on EKS with one Helm command. (If you already cloned the repo earlier for local testing, just navigate to EKS/helm.)

Note: The Helm chart is configured to use pre-built Docker images for all services (hosted on Docker Hub), so you do not need to build or push any images manually. The default values should work out-of-the-box for a demo deployment.

**2. Install the Helm Chart on EKS**
We’ll deploy all components into a dedicated Kubernetes namespace (for cleanliness). Create the namespace and install the Helm release:

```
kubectl create namespace robot-shop
helm install robot-shop . -n robot-shop
```

This tells Helm to install the chart in the current directory (.) into the robot-shop namespace, with the release name robot-shop. Helm will then create all the necessary objects: deployments for each microservice, stateful sets for database components, services for internal communication, etc. After running the above commands, Helm’s output should indicate it created a number of Kubernetes resources. Indeed, this single step provisions “all the services, deployments, and configurations necessary for the Robot Shop application to run inside your EKS cluster.”

Give it a minute to let Kubernetes pull the container images and start up all the pods. You can run the following to check the status of the pods:

```
kubectl get pods -n robot-shop
```

All pods should eventually show as Running (or Completed for any one-time jobs). If any pods are in CrashLoopBackOff, you may need to check logs, but in a normal scenario they should start correctly since we’re using known-good images.

**3. Configure Ingress and Load Balancer (Expose the Application)**
At this point, the application is running internally in the cluster, but we need to make it accessible from the outside. To do that, we’ll create a Kubernetes Ingress resource which the AWS ALB Controller will pick up and create an external Application Load Balancer for. The Helm chart directory provides an ingress.yaml file with the necessary Ingress configuration (it likely defines path routing to the web service, which is the frontend).

Apply the ingress manifest:

```
kubectl apply -f ingress.yaml
```

This creates an Ingress resource (in the robot-shop namespace) that maps incoming HTTP requests to the web service of our application. The AWS Load Balancer Controller we installed will notice this and immediately start provisioning an ALB to satisfy the Ingress. Within a few minutes, a new ALB will be created in your AWS account (you can check the EC2 > Load Balancers section in the AWS console to watch its progress). The Ingress controller will also create a corresponding Kubernetes Service of type LoadBalancer under the hood, but you primarily interact with the Ingress itself.

It may take 5-10 minutes for the Application Load Balancer to be fully provisioned and in an “active” state. You can verify that the Ingress has been assigned an address by running:

```
kubectl get ingress -n robot-shop
```

Once the STATUS shows an address (which will be an AWS ALB DNS name), your application is available on that URL. For example, you might see an address like k8s-robotshop-ABCDEFGHI.us-east-1.elb.amazonaws.com. You can also get the ALB’s DNS by checking the AWS console.

**4. Access the Application**
When the ALB is ready, copy its DNS name and try opening it in your web browser (it will be serving on HTTP port 80 by default). You should see the Robot Shop storefront load in your browser, confirming that the frontend service is reachable. From there, you can navigate the site: create a user account, browse products, add items to your cart, and perform a checkout. These actions will trigger the various microservices (user, catalogue, cart, order, payment, etc.) to work together. If all went well, the order checkout will complete and you’ll get an order confirmation, indicating the full path through the three tiers is functioning.

At this stage, you have successfully deployed the full microservices-based three-tier application on AWS EKS. The ALB is handling external traffic and routing through the Ingress to the web frontend, which in turn calls internal services. Kubernetes is managing the scaling and networking of all those components behind the scenes.

Tip: You can also verify all Services are properly exposed by running kubectl get svc -n robot-shop. Most services are ClusterIP (internal only), but the ingress/ALB provides the external access point. If you described the Ingress (kubectl describe ingress -n robot-shop), you would see annotations that the ALB controller added (like AWS load balancer ARNs, etc.), and the rules mapping /* path to the web service.

**5. (Optional) Clean Up Resources**
If you are done experimenting, remember to delete the AWS resources to avoid ongoing charges. The easiest way is to delete the EKS cluster, which will also delete node instances and load balancers:

```
eksctl delete cluster --name demo-cluster-three-tier-1 --region us-east-1
```

This will tear down the entire cluster and associated resources (it may take a few minutes to complete). Ensure that the ALB, EC2 instances, and any EBS volumes created are removed. Double-check AWS console for any remaining pieces to delete manually if needed. Only do this clean-up step when you no longer need the cluster or application running.

Optional: Load Testing and Monitoring
After deployment, you might want to generate some traffic on the application or integrate monitoring to observe its behavior. Stan’s Robot Shop comes with a couple of additional components for these purposes.

Load Generation: The repository includes a utility in the load-gen directory to simulate user traffic to the app. This load generator is a Python/Locust-based tool that can continuously hit the various endpoints of the Robot Shop (simulating users browsing and purchasing). It is not deployed by default. You can run it manually in a Docker container, or even deploy it into your EKS cluster as a Job/Deployment. (The repo provides an example Kubernetes descriptor for running the load generator in the K8s/ folder.) Using this, you can apply load to the system and test its resilience. For example, Locust can simulate multiple concurrent users adding items to carts and checking out.

Application Monitoring & Metrics: The microservices are instrumented for observability. In fact, the application was originally built to showcase Instana APM, and each service has Instana tracing agents included (if you have an Instana instance and configure the agent key, you could see end-to-end distributed traces in Instana’s dashboard). For a more open-source approach, certain services also expose Prometheus metrics endpoints. Notably, the cart service and payment service have HTTP endpoints at /metrics which provide Prometheus-formatted metrics. These include counters such as the total number of items added to carts and purchased, and histograms of cart sizes and values. If you set up a Prometheus server (either in-cluster or via AWS Managed Prometheus) to scrape these endpoints, you can monitor the app’s performance and usage. You could then visualize the data with Grafana or CloudWatch. Additionally, you can monitor cluster-level metrics (pod CPU/memory, etc.) via CloudWatch Container Insights or Prometheus as well.

By combining load generation and monitoring, you can observe how the system behaves under load – for example, watching the metrics for database usage or throughput as Locust drives traffic. This can be a great learning exercise in scaling and performance tuning for a microservices architecture.


