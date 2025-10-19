## 🗂️ Example — EFS StorageClass YAML


```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0a1b2c3d4e
  directoryPerms: "777"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
reclaimPolicy: Delete
volumeBindingMode: Immediate
```


Field-by-Field Breakdown
1️⃣ provisioner: efs.csi.aws.com

This tells Kubernetes which CSI driver to use.
Here it’s the AWS EFS CSI driver, which knows how to:

Create access points

Mount EFS filesystems to pods

Manage directories and permissions

Think of this as “the plugin that understands how to talk to EFS.”

2️⃣ parameters: — Core config for EFS

Each field here controls how the EFS storage behaves 👇

🧩 fileSystemId: fs-0a1b2c3d4e

This is the EFS FileSystem ID created in AWS (through console or Terraform).

It tells Kubernetes which EFS filesystem to use for provisioning volumes.

💡 Example: If your team has one shared EFS per cluster, you’ll reuse this ID across many PVs.

directoryPerms: "777"

Sets permissions for the subdirectory inside EFS (when it’s created).

"777" = read/write/execute by everyone (owner, group, others).

Useful when you want multiple pods or users to have access.


### Use of file system id and why we dont have such thing in ebs??


🧩 The Core Difference:
EFS = pre-created, shared filesystem
EBS = dynamically created, one-time disk

