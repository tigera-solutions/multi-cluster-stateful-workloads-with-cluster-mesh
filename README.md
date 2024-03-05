# Easily connect stateful workloads across multiple Kubernetes clusters with Calico Cluster Mesh

Calico Cluster Mesh extends Kubernetes' inherent capabilities, providing seamless service discovery across multiple Kubernetes clusters. This advanced feature allows Kubernetes services, including headless services, to discover and connect with each other across cluster boundaries without the need for an additional control plane, such as a Service Mesh.

This example outlines the setup of two AWS EKS clusters with cross-region connectivity. Each cluster is placed within its own Virtual Private Cloud (VPC), and these VPCs are connected to allow direct network communication between the clusters using VPC peering. The configuration ensures that EKS cluster nodes in one VPC can communicate with cluster nodes in the other VPC.

The EKS clusters are configured with Calico Cluster Mesh, enabling direct, low-latency communication between clusters. This allows services in different clusters to discover and connect seamlessly, simplifying cross-cluster interactions and enhancing network efficiency without the need for additional networking layers or external routing mechanisms.


## Solution Overview

## Walk Through

We'll use Terraform, an infrastructure-as-code tool, to deploy this reference architecture automatically. We'll walk you through the deployment process and then demonstrate how to utilize [Calico Cluster Mesh on AWS](https://docs.tigera.io/calico-enterprise/latest/multicluster/federation/overview)

### Prerequisites:

First, ensure that you have installed the following tools locally.

1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [kubectl](https://Kubernetes.io/docs/tasks/tools/)
3. [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
3. [jq](https://jqlang.github.io/jq/download/)

### Step 1: Checkout and Deploy the Terraform Blueprint

#### 1. Clone the Terraform Blueprint
Make sure you have completed the prerequisites and then clone the Terraform blueprint:
```sh
git clone https://github.com/tigera-solutions/multi-cluster-stateful-workloads-with-cluster-mesh.git
```

#### 2. Navigate to the AWS Directory
Switch to the `aws` subdirectory:
```sh
cd aws
```

#### 3. Customize Terraform Configuration
Optional: Edit the [terraform.tfvars](aws/terraform.tfvars) file to customize the configuration.

Examine [terraform.tfvars](aws/terraform.tfvars).

```sh
region1         = "us-east-1"
region2         = "us-west-2"
vpc1_cidr       = "10.0.0.0/16"
vpc2_cidr       = "10.1.0.0/16"
cluster1_name   = "iad"
cluster2_name   = "pdx"
cluster_version = "1.27"
instance_type   = "m5.xlarge"
desired_size    = 3
ssh_keyname     = "your-ssh-keyname"
pod_cidr1       = "192.168.1.0/24"
pod_cidr2       = "192.168.2.0/24"
calico_version  = "v3.26.4"
calico_encap    = "VXLAN"
```

#### 4. Deploy the Infrastructure
Initialize and apply the Terraform configurations:
```sh
terraform init
terraform apply
```

Enter `yes` at command prompt to apply

#### 5. Update Kubernetes Configuration
Update your kubeconfig with the EKS cluster credentials as indicated in the Terraform output:

```sh
aws eks --region <REGION1> update-kubeconfig --name <CLUSTER_NAME1> --alias <CLUSTER_NAME1>
aws eks --region <REGION2> update-kubeconfig --name <CLUSTER_NAME2> --alias <CLUSTER_NAME2>
```

#### 6. Verify Calico Installation
Check the status of Calico in your EKS cluster:
```sh
kubectl --context iad get tigerastatus
kubectl --context pdx get tigerastatus
```

## Step 2: Link Your EKS Cluster to Calico Cloud

#### 1. Join the EKS Cluster to Calico Cloud
Join your EKS cluster to [Calico Cloud](https://www.calicocloud.io/home) as illustrated:

https://github.com/tigera-solutions/multi-cluster-stateful-workloads-with-cluster-mesh/assets/101850/cae8ffac-bc65-4b20-88bb-180168fdcacc

#### 2. Verify the Cluster Status
Check the cluster status:
```sh
kubectl --context iad get tigerastatus
kubectl --context pdx get tigerastatus
```

#### 3. Update the Felix Configuration
Set the flow logs flush interval:
```sh
kubectl --context iad patch felixconfiguration default --type='merge' -p '{
  "spec": {
    "dnsLogsFlushInterval": "15s",
    "l7LogsFlushInterval": "15s",
    "flowLogsFlushInterval": "15s",
    "flowLogsFileAggregationKindForAllowed": 1
  }
}'
kubectl --context pdx patch felixconfiguration default --type='merge' -p '{
  "spec": {
    "dnsLogsFlushInterval": "15s",
    "l7LogsFlushInterval": "15s",
    "flowLogsFlushInterval": "15s",
    "flowLogsFileAggregationKindForAllowed": 1
  }
}'
```

## Step 3: Cluster Mesh for AWS Elastic Kubernetes Service

#### 1. Create the Cluster Mesh

Run the [setup-mesh.sh](setup-mesh.sh) script:
```sh
cd ..
sh setup-mesh.sh
```

The `setup-mesh.sh` script automates the creation of a Calico cluster mesh as outlined in the [Tigera documentation](https://docs.tigera.io/calico-cloud/multicluster/overview), enabling secure and efficient connections between multiple Kubernetes clusters. Below is a breakdown of the specific Kubernetes resources it creates and configures:

1. **In the source cluster**, it:
   - Applies Calico federation manifests to install federation roles, rolebindings, and a service account needed for cross-cluster communication.
   - Creates a secret that stores the service account token. This token ensures secure connections between clusters by providing authentication and authorization.

2. **Generates a kubeconfig file** using the service account token. This file contains all necessary details (like the cluster API server address and credentials) for secure access to the source cluster.

3. **In the destination cluster**, it:
   - Creates a secret that includes the kubeconfig from the source cluster. This enables the destination cluster to securely communicate with the source cluster.
   - Configures a `RemoteClusterConfiguration` resource, which is used to manage the mesh connection settings and policies.
   - Applies specific RBAC roles and role bindings to allow designated components access to the secret, ensuring they can establish and maintain secure cross-cluster communication.
  
https://github.com/tigera-solutions/multi-cluster-stateful-workloads-with-cluster-mesh/assets/101850/ac46c8d8-85cb-4a47-95e2-9f8b185bd6a0

## Validate the Deployment and Review the Results

#### 1. Confirm Calico Cluster Mesh enabled clusters are in-sync
Check logs for remote cluster connection status:
```sh
kubectl --context iad logs deployment/calico-typha -n calico-system | grep "Sending in-sync update"
kubectl --context pdx logs deployment/calico-typha -n calico-system | grep "Sending in-sync update"
```

```
2024-02-27 01:51:06.156 [INFO][13] wrappedcallbacks.go 487: Sending in-sync update for RemoteClusterConfiguration(pdx)
2024-02-27 01:51:03.300 [INFO][13] wrappedcallbacks.go 487: Sending in-sync update for RemoteClusterConfiguration(iad)
```

You should see similar messages for each of the clusters in your cluster mesh.

#### 2. Deploy Statefulsets and Headless Services
Return to the project root and apply the manifests:
```sh
kubectl --context iad apply -f multi-cluster-rs-iad.yaml
kubectl --context iad apply -f netshoot.yaml
kubectl --context pdx apply -f multi-cluster-rs-pdx.yaml
kubectl --context pdx apply -f netshoot.yaml
```

#### 3. Implement Calico Federated Services for Calico Cluster Mesh
Test the configuration of each Service:

```sh
kubectl --context pdx get svc
```

```sh
kubectl --context pdx exec -it netshoot -- ping -c 1 multi-cluster-rs-pdx
kubectl --context pdx exec -it netshoot -- ping -c 1 multi-cluster-rs-iad
```

```sh
kubectl --context iad get svc
```

```sh
kubectl --context iad exec -it netshoot -- ping -c 1 multi-cluster-rs-iad
kubectl --context iad exec -it netshoot -- ping -c 1 multi-cluster-rs-pdx
```

#### 4. Cleanup

To teardown and remove the resources created in this example:

```sh
terraform state rm helm_release.calico_cluster1
terraform state rm helm_release.calico_cluster2
terraform destroy --auto-approve
```
