## What is Amazon EKS

EKS = Elastic Kubernetes Service → AWS’s managed Kubernetes control plane.


AWS runs and manages:

The control plane (API server, scheduler, controller manager, etc.)

The etcd database (stores cluster state)

Control plane auto-scales, patches, upgrades, and is highly available

You manage:

Worker nodes (EC2, Fargate, or both)

Networking

Add-ons (like AWS Load Balancer Controller, CoreDNS, kube-proxy, etc.)


3️⃣ Control plane vs. data plane

| Component            | Managed by AWS? | Description                                   |
| -------------------- | --------------- | --------------------------------------------- |
| API Server           | ✅ Yes           | Accepts and processes all Kubernetes commands |
| etcd                 | ✅ Yes           | Stores cluster state                          |
| Scheduler            | ✅ Yes           | Decides where pods run                        |
| Worker Nodes         | ❌ You manage    | Run the actual containers                     |
| kubelet / kube-proxy | ❌ You manage    | Communicate with control plane                |


### Control plane vs Data plane

🎯 The Big Picture


Think of Kubernetes (and EKS) like an airport.

✈️ Planes = your pods (apps/containers)

👨‍✈️ Pilots = kubelet on each node

🛫 Control Tower = control plane

🏗️ Runway, terminals, fuel stations = data plane


### 🧠 CONTROL PLANE (The Brain 🧠)


The control plane is the brain or control tower of Kubernetes.It decides what should happen in your cluster — but it doesn’t actually run your apps.

Key components:

| Component              | Role                                         | Analogy                                           |
| ---------------------- | -------------------------------------------- | ------------------------------------------------- |
| **API Server**         | Accepts all commands (like `kubectl apply`)  | Airport control tower receiving flight plans      |
| **Scheduler**          | Decides *where* pods should run (which node) | Assigns which runway each plane uses              |
| **Controller Manager** | Ensures reality matches what you declared    | Makes sure flights actually take off              |
| **etcd**               | Database storing cluster state               | Central flight log — knows every plane’s position |

In EKS, AWS runs and manages this control plane for you.
You don’t see or manage these servers — AWS keeps them HA (multi-AZ), secure, and patched.

### ⚙️ DATA PLANE (The Muscles 💪)

The data plane is where the actual work happens — where your containers (pods) really run.

This includes:

Worker nodes (EC2 instances or Fargate)→ each node runs:

kubelet — agent that talks to the control plane

kube-proxy — handles pod-to-pod networking

container runtime (e.g., containerd, Docker)


The data plane:

Executes the control plane’s instructions

Hosts pods and manages CPU, RAM, networking, storage

+---------------------------------------------+
|                 CONTROL PLANE                |
|---------------------------------------------|
| API Server | Scheduler | Controller Manager |
| etcd (cluster database)                     |
| (Managed by AWS in EKS)                     |
+---------------------------------------------+

                 ⬇️ (communicates via API)

+---------------------------------------------+
|                 DATA PLANE                  |
|---------------------------------------------|
| Worker Nodes (EC2 or Fargate)               |
|  - kubelet (agent)                          |
|  - kube-proxy (networking)                  |
|  - Pods (containers)                        |
|  (Managed by you)                           |
+---------------------------------------------+


🧩 Example

Let’s say you run:
```
kubectl apply -f nginx.yaml
```

Here’s what happens:

1️⃣ You tell the API Server → “I want 3 nginx pods.”
2️⃣ The Scheduler decides which nodes can run them.
3️⃣ The Controller Manager ensures 3 pods exist.
4️⃣ The Worker nodes (data plane) actually start those nginx pods.
5️⃣ If one node dies → the control plane notices → reschedules pods elsewhere.


### but how kubelet needs my involvement??


You don’t install kubelet manually.
When you create worker nodes in EKS, AWS provides EKS-optimized AMIs (Amazon Machine Images).
These come with:
- kubelet
- kube-proxy
- container runtime (containerd)
- AWS IAM authenticator

So, AWS prepares the image,but you decide:

- which AMI to use (and its version)
- when to upgrade it
- when to restart or scale nodes
- how to configure its resources

### What “managing kubelet” means in practice

| Task                | How you do it                                              | What it affects                            |
| ------------------- | ---------------------------------------------------------- | ------------------------------------------ |
| **Node creation**   | Launch node groups via `eksctl`, Terraform, or AWS console | Starts kubelet automatically on new EC2s   |
| **Node scaling**    | Configure Cluster Autoscaler                               | Adds/removes kubelets automatically        |
| **Upgrades**        | Replace nodes with new AMIs (new kubelet versions)         | Keeps kubelet version aligned with cluster |
| **Configuration**   | Modify user data or systemd flags                          | Tune kubelet behavior (rarely needed)      |
| **Monitoring**      | Check node health via `kubectl get nodes` or CloudWatch    | Confirms kubelet is alive and reporting    |
| **IAM permissions** | Attach IAM roles to node groups                            | Controls what kubelet can access in AWS    |



### Then why is it “managed by you”?

Because even though AWS gives you the software,
you own the EC2 instance where it runs.

That means:

- If your node crashes → you fix or replace it.
- If you want a new Kubernetes version → you upgrade the node AMI (which updates kubelet).
- If kubelet stops sending updates → you restart or recreate that node.
-AWS won’t log in to your EC2 to do that for you.

So “you manage kubelet” really means:

You manage the machine that kubelet lives on, not the kubelet code itself.

## Key AWS components involved

- IAM → permissions for nodes, service accounts, and users
- VPC → networking backbone of the cluster
- EC2 or Fargate → worker nodes (compute layer)
- ECR (Elastic Container Registry) → where you store Docker images 
- CloudWatch → logs and metrics
- EFS / EBS / S3 → persistent storage options
- ALB / NLB → ingress / load balancing for services


### ⚙️ 1️⃣ IAM (Identity and Access Management)

What it is:

IAM controls who can do what in AWS — permissions for users, roles, and services.

How it connects to EKS:

In EKS, permissions exist at 3 levels:


| Level                        | Example                             | Controlled by                       |
| ---------------------------- | ----------------------------------- | ----------------------------------- |
| **Cluster access (kubectl)** | Which IAM user can access EKS API   | IAM → `aws-auth` ConfigMap          |
| **Node permissions**         | Nodes pulling images, creating ENIs | IAM Role for EC2 node               |
| **Pod permissions**          | Pods accessing S3, DynamoDB, etc.   | IAM Role for Service Account (IRSA) |

🧠 Why it matters:

Without proper IAM:

- Your pods can’t access AWS services (S3, DynamoDB, Secrets Manager, etc.)
- Nodes can’t join the cluster
- You might give too many permissions (security risk)


✅ Real example:

If your app needs to read from S3:

Create an IAM role with s3:GetObject

Link it to the pod’s service account (IRSA)

That pod can now securely access S3 without storing AWS credentials

### 🌐 2️⃣ VPC (Virtual Private Cloud)

💡 What it is:

VPC = your private network in AWS.

It defines:

- Subnets (public/private)
- Route tables
- Security groups
- NAT Gateways

How it connects to EKS:

EKS clusters live inside your VPC.

- Control plane connects to your VPC via ENIs (Elastic Network Interfaces)
- Pods get IPs from your subnets (via the CNI plugin)
- Load balancers attach to subnets to expose services


Why it matters:

If your VPC networking is wrong, your pods:

- Can’t talk to the internet
- Can’t reach each other across nodes
- Can’t mount EFS or call AWS APIs


✅ Real example:

You have:

Private subnets → where EKS worker nodes live

Public subnets → for ALB to expose your app to the internet



3️⃣ How AWS Load Balancer Controller works

Step by step:

- You create a Service of type LoadBalancer or an Ingress in Kubernetes.
- The AWS Load Balancer Controller notices it.
- It creates an ALB in the public subnet (based on tags).
- It attaches ENIs (network interfaces) to the public subnet.
- Traffic from the internet hits the ALB.
- The ALB forwards traffic into your private subnets, to worker nodes, and finally to pods.


### 🖥️ 3️⃣ EC2 or Fargate — Worker Nodes

💡 What they are:

These are where your containers actually run (the data plane).

How they connect to EKS:

Each node runs kubelet, which talks to the EKS control plane.

You can choose between:

- EC2 nodes → full control, SSH access
- Fargate → serverless pods (no node management)

## 📦 4️⃣ ECR (Elastic Container Registry)

What it is:

Private Docker image registry on AWS (similar to Docker Hub).

How it connects to EKS:

You push your built images here (docker push ...)

EKS worker nodes pull images from ECR when creating pods

Why it matters:

- ECR integrates with IAM (secure, no password-based access)
- Reduces latency (since it’s in-region)
- You can use lifecycle policies to clean up old images

```
Pod → Node → NAT Gateway → ECR → Pull image
```

### 📊 5️⃣ CloudWatch

What it is:

AWS’s centralized monitoring & logging service.

How it connects to EKS:

You can use CloudWatch for:

- Cluster metrics (CPU, memory via CloudWatch Container Insights)
- Logs (via FluentBit/FluentD DaemonSets)
- Alarms & dashboards



Why it matters:

Helps you find out which pod/node crashed

You can set alerts for pod OOMs or node failures

Critical for production observability


## EKS Node Groups & Autoscaling — Deep Dive

In Kubernetes (and EKS), pods scale dynamically based on demand — but nodes (the machines running those pods) also need to scale.
That’s where Node Groups, Cluster Autoscaler, HPA, and Karpenter come in.

### ⚙️ Node Groups Overview

A node group is a collection of EC2 instances that act as worker nodes in your EKS cluster.
All nodes in a group:

- share the same AMI, instance type, and configuration
- join the same cluster automatically at startup
- scale in/out together

EKS supports two main types:

| Type                               | Managed by AWS? | Description                                                                           | Typical Use Case                     |
| ---------------------------------- | --------------- | ------------------------------------------------------------------------------------- | ------------------------------------ |
| **Managed Node Group (MNG)**       | ✅ Yes           | AWS automates provisioning, lifecycle (upgrades, draining, replacing unhealthy nodes) | Recommended for most users           |
| **Self-Managed Node Group (SMNG)** | ❌ No            | You provision EC2 instances yourself (via Terraform, CloudFormation, etc.)            | When you need custom AMIs or configs |

### 🧠 Managed Node Groups (MNG)

When you create a Managed Node Group:
- AWS launches EC2 instances using EKS-optimized AMIs.
- The nodes auto-register with the control plane.
- AWS handles draining & replacement during upgrades.
- Integrates natively with Cluster Autoscaler and Karpenter.

Key benefits:

- Simplified lifecycle management
- Auto-replacement of unhealthy nodes
- Easy scaling via console or eksctl

```
eksctl create nodegroup \
  --cluster my-eks-cluster \
  --name ng-app \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 6 \
  --managed
```

### 🧰 Self-Managed Node Groups

If you need custom AMIs or very specific OS/network/storage configs,
you can create nodes manually using:

- Launch Templates
- Auto Scaling Groups (ASGs)
- EC2 User Data scripts

```
/etc/eks/bootstrap.sh my-eks-cluster
```

### 📈 Cluster Autoscaler (CA)

The Cluster Autoscaler adjusts the number of nodes in your cluster automatically.

What it does

- Watches for unschedulable pods (pods that can’t find room on existing nodes)
- Scales up the node group (adds EC2 instances)
- Scales down when nodes are underutilized

Cluster Autoscaler interacts with:

AWS Auto Scaling Groups (ASGs) or

Managed Node Groups

How it works

- You deploy it as a pod in your cluster.

- It monitors pending pods and node utilization.

- Uses AWS APIs to modify the node group size.


### ⚡ 4️⃣ Karpenter — Modern Node Autoscaling

Karpenter is AWS’s next-gen, open-source autoscaler, designed to replace Cluster Autoscaler.

🚀 Why it’s faster

- Doesn’t depend on ASGs.
- Launches nodes directly via the EC2 API.
- Responds in seconds (not minutes).
- Automatically picks the best instance type for workload.


# Karpenter vs cluster autoscaler


## 🚀 Cluster Autoscaler (the old way)
🧩 How it works

Cluster Autoscaler (CA) was designed when Kubernetes clusters ran on Auto Scaling Groups (ASGs).

So in EKS, when new pods can’t be scheduled:

- The CA notices there are pending pods.

- It tells the ASG: “Please add 1 more node.”

- The ASG launches a new EC2 instance.

- That instance boots, joins the cluster → pods get scheduled.

## Problem: It’s indirect

- CA doesn’t launch EC2s itself — it must ask the ASG to do it.
- ASGs are slow — they check health, scaling policies, cooldowns.
- You must predefine instance types & counts in the ASG.

## ⚡ Karpenter (the modern way)

Karpenter skips the middleman — it talks directly to the EC2 API.
No Auto Scaling Groups, no predefined instance types.

When pods can’t be scheduled:

- Karpenter sees the pending pods.
- It analyzes their needs (CPU, memory, GPU, zone, taints, etc.).
- It directly calls EC2’s API to launch the best-fitting instance.
- The new node joins → pods start running — all in seconds.


# 🌐 2️⃣ EKS Networking Deep Dive

Kubernetes networking is already complex — then AWS adds its VPC and ENI concepts.
EKS bridges those worlds using the AWS VPC CNI plugin.

## ⚙️ AWS VPC CNI Plugin (Container Network Interface)

CNI = Container Network Interface → defines how pods get network connectivity.
EKS uses the AWS VPC CNI plugin, developed and maintained by AWS.

🔍 What it does

The VPC CNI plugin assigns:

- VPC-native IP addresses (from your subnets)

- Direct ENI (Elastic Network Interface) attachments to your pods.

## 🧩 How it works under the hood

Each EC2 node (worker) has:

- 1 primary ENI (attached at launch)

- Some number of secondary ENIs that can host multiple pod IPs

The CNI plugin:

- Requests ENIs from the VPC subnet.
- Attaches them to the EC2 node.
- Allocates IPs from those ENIs to the pods running on that node.

### lets dig deep buddy

## 🧠 First: What problem is the CNI solving?

When you create a pod in Kubernetes, that pod needs a network identity — an IP address — so it can:

- communicate with other pods,
- talk to the internet,
- talk to AWS services (like S3, RDS, etc).

So the CNI plugin decides how and from where to get that IP address.

## 🧩 Let’s imagine it visually

🏠 Your VPC subnet
Think of your subnet like a neighborhood of IP houses — say you have:
```
10.0.0.0/24  →  256 possible IPs
```
Some IPs go to:

- EC2 instances (nodes)
- Load balancers
- Pods

### ⚙️ When you create an EC2 node (a worker node)
That EC2 instance automatically gets: 1 Primary ENI (Elastic Network Interface) — this is like the node’s “main network card.”

```
Node1 ENI → 10.0.0.5
```

### 🔌 Now comes the magic: Secondary ENIs

Each EC2 instance type supports multiple ENIs.
Each ENI can have multiple private IPs.


why do we need secondary eni??

Your node can run many pods, and each pod needs a unique IP.

But:

The primary ENI has only one private IP (or a few).

We need more IPs to assign to all those pods.

So the AWS VPC CNI plugin says:

“Hey AWS, please give me more network interfaces (ENIs)
so I can get more IPs to hand out to pods.”

AWS attaches secondary ENIs to your EC2 node.
Each secondary ENI:

belongs to the same subnet,

has multiple private IP addresses,

can be used by the VPC CNI to assign IPs to pods.


🧩 Example

Suppose:

Subnet = 10.0.0.0/24

Node (EC2) = 10.0.0.5 (primary ENI)

Secondary ENI attached = eni-12345

Secondary ENI has these private IPs: 10.0.0.10–10.0.0.14

| Pod   | Assigned IP | From where?   |
| ----- | ----------- | ------------- |
| Pod-A | 10.0.0.10   | Secondary ENI |
| Pod-B | 10.0.0.11   | Secondary ENI |
| Pod-C | 10.0.0.12   | Secondary ENI |


📊 Why it’s awesome

✅ Simple routing — no overlays, no extra hops
✅ Native VPC security controls
✅ Pod-to-pod, pod-to-service, and pod-to-AWS-service networking “just works”
✅ Lower latency


⚠️ But there are limits

Each EC2 instance type has a maximum number of ENIs and IPs per ENI.

Example (us-west-2):

| Instance Type | Max ENIs | IPs per ENI | Max Pods |
| ------------- | -------- | ----------- | -------- |
| t3.medium     | 3        | 6           | 17       |
| m5.large      | 3        | 10          | 29       |
| m5.2xlarge    | 4        | 30          | 110      |


```
VPC
 ├─ Subnet (10.0.0.0/24)
 │   ├─ EC2 Node (10.0.0.5)
 │   │   ├─ ENI1 (primary)
 │   │   ├─ ENI2 (secondary)
 │   │   │   ├─ Pod A (10.0.0.6)
 │   │   │   ├─ Pod B (10.0.0.7)
 │   │   │   ├─ Pod C (10.0.0.8)
```

## 🔐 Security Groups for Pods (SGP)

By default:

All pods on a node share the node’s security group.

If you need fine-grained network isolation (e.g., DB pods can’t talk to web pods), you can use Security Groups for Pods.

🧱 How it works

Uses ENI Trunking to attach multiple security groups to a node.

CNI plugin assigns specific SGs to specific pods.

You annotate pods with SGs.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-db
  annotations:
    k8s.v1.cni.cncf.io/networks: vpc-cni
    vpc.amazonaws.com/pod-eni: "true"
    vpc.amazonaws.com/security-groups: sg-0abcd1234ef56789
```

### lets dig deep

🧩 What are Security Groups?

A Security Group (SG) is like a firewall in AWS.
It decides which traffic can go in or out of a resource (like an EC2, RDS, or ENI).

Example:
sg-web allows TCP 80/443 (HTTP/HTTPS) from the internet.
sg-db allows TCP 5432 (PostgreSQL) only from sg-web.


🧩 What’s the default situation in EKS?


By default:

Each node (EC2 instance) in EKS has one security group, say sg-node.

All pods on that node share that same security group.


That means all pods on that node:

Can talk to everything sg-node allows.

Have no separate network firewall rules between them.


So if you put both a web pod and a database pod on the same node — they both use sg-node, and they can talk to each other freely, even if you didn’t want that.


🧱 Problem

You might want:

Frontend pods → allowed from internet (port 80)

Backend pods → only internal traffic

Database pods → only allow backend pods on port 5432

### 🧠 Solution — Security Groups for Pods (SGP)

SGP lets you attach a separate AWS security group directly to individual pods, not just to nodes.


This means:

Each pod can have its own firewall rules, just like an EC2 instance.

So your database pod can have sg-db (only backend access),
and your frontend pod can have sg-web (public access),
even if they’re running on the same node!


### ⚙️ How AWS makes that possible (ENI Trunking)

ENIs (Elastic Network Interfaces) are like network cards attached to EC2s.
But there’s a limit on how many you can attach to a node — normally just a few.

So AWS invented ENI Trunking for this.

Think of it like this 👇

```
[EC2 Node]
   │
   ├── Trunk ENI  ← main “pipe” that carries traffic
   │
   ├── Branch ENI (for Pod A) → Security Group = sg-web
   ├── Branch ENI (for Pod B) → Security Group = sg-db
   └── Branch ENI (for Pod C) → Security Group = sg-backend
```

Each Branch ENI belongs to one pod, and it can have its own security group.

The VPC CNI plugin does this automatically when you annotate your pod.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  annotations:
    vpc.amazonaws.com/pod-eni: "true"
    vpc.amazonaws.com/security-groups: sg-0abcd1234ef56789
spec:
  containers:
    - name: postgres
      image: postgres
```

🧩 Meaning:

vpc.amazonaws.com/pod-eni: "true" → create a dedicated ENI for this pod.

vpc.amazonaws.com/security-groups: → attach this SG to that ENI.

Now that pod:

Gets its own IP and network interface.

Has its own SG — not the node’s SG.


# upcoming

4️⃣ EKS Add-ons

Built-in AWS add-ons:

CoreDNS, kube-proxy

VPC CNI

Optional:

AWS Load Balancer Controller

Cluster Autoscaler

Metrics Server

External Secrets Operator

FluentBit (for logs)

Include how to install them via:

eksctl utils associate-iam-oidc-provider
eksctl create addon --name vpc-cni

5️⃣ Security in EKS

IAM Roles for Service Accounts (IRSA) – least-privilege pod access

Network Policies – control pod-to-pod communication

Secrets management via AWS Secrets Manager / External Secrets

EKS Pod Identity (newer alternative to IRSA)

Encryption at rest (KMS + etcd)

Security scanning with tools like Trivy or Kube-bench

6️⃣ EKS Cluster Access & Authentication

How aws-auth ConfigMap maps IAM users/roles → Kubernetes RBAC

Use of kubectl, aws eks update-kubeconfig

Fine-grained RBAC for namespaces, service accounts, etc.

7️⃣ EKS Upgrades & Versioning

AWS manages control plane upgrades

You manage node AMI upgrades (kubelet version)

Rolling upgrades & draining strategies for production

8️⃣ Observability & Logging

CloudWatch Container Insights for metrics

FluentBit / FluentD for centralized logging

Prometheus + Grafana for in-cluster monitoring

AWS Distro for OpenTelemetry (ADOT) for tracing

9️⃣ Deployment Workflows

kubectl / kustomize / Helm for manifests

ArgoCD or Flux for GitOps

CI/CD Integration – using GitHub Actions, CodePipeline, etc.

🔟 Cost Optimization & Best Practices

Use spot instances in node groups

Use right-sizing metrics from CloudWatch / Cost Explorer

Cluster Autoscaler and Karpenter tuning

Cleanup unused EBS volumes, ENIs, or old images in ECR
