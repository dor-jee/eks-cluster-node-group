# eks-cluster-node-group

![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Cluster%20Autoscaler-326CE5?logo=kubernetes&logoColor=white)
![CloudFormation](https://img.shields.io/badge/IaC-CloudFormation%20(VPC%20only)-FF4F8B?logo=amazonaws&logoColor=white)
![Region](https://img.shields.io/badge/region-eu--central--2%20(Z%C3%BCrich)-informational)


Teams running variable workloads on Kubernetes need capacity that grows and shrinks with demand — manually sized clusters either waste money sitting idle or fall over under load. Built as a practice project to learn how a production-style EKS setup fits together end to end, this repository stands up a minimal but complete platform: a dedicated VPC, a managed node group, and a Cluster Autoscaler that adds nodes when pods can't schedule and removes them once load drops, with AWS access through OIDC rather than static credentials.

Every component — network, IAM, cluster, autoscaler — was configured explicitly through the console and kubectl, without IaC, so each decision and its tradeoff had to be understood rather than inherited from a module.

Sole engineer: designed, built and documented the whole stack; documentation drafted with AI assistance and every claim verified against the actual build.

**What you get:** an EKS control plane in a dedicated VPC with 4 subnets (public and private), a managed node group of 2 `t3.small` On-Demand nodes, a Cluster Autoscaler wired to AWS through OIDC, and a sample nginx deployment exposed through a `LoadBalancer` service.

**Verified state:** 2 of 2 nodes `Ready`, the nginx pod `1/1 Running`, the Classic Load Balancer reports 2 of 2 instances in service, the autoscaler pod was still `Running` with 0 restarts after 6h15m, and a full scale-up/down cycle was verified (see [Infrastructure](#infrastructure)).

---

## Architecture


```mermaid
flowchart LR
    user([kubectl / browser]) --> elb[Classic Load Balancer<br/>internet-facing]
    user --> api[EKS API endpoint<br/>public + private]

    subgraph vpc[eks-worker-node-vpc-stack-VPC]
        elb --> nginx[nginx pods]
        subgraph nodes[Node group: eks-node-group]
            nginx
            ca[cluster-autoscaler<br/>kube-system]
        end
    end

    api --> cp[EKS control plane<br/>eks-cluster-test]
    cp --- nodes
    ca -- OIDC / sts.amazonaws.com --> iam[IAM role<br/>EKSServiceAccountRole]
    iam --> asg[EC2 Auto Scaling group]
    cp --> obs[CloudWatch + Prometheus metrics]
```

---

## Tech stack

| Component | Tool | Why |
|---|---|---|
| Kubernetes control plane | Amazon EKS | Managed control plane, no etcd or API server to operate |
| Networking | AWS VPC via CloudFormation template | Default VPC is not suited for EKS; the AWS template gives public and private subnets out of the box |
| Worker nodes | EKS managed node group (Amazon Linux 2023, `t3.small`, On-Demand) | One node group object instead of creating each EC2 instance by hand |
| Autoscaling | Kubernetes Cluster Autoscaler | Scales the node group up and down through the AWS Auto Scaling API |
| Auth to AWS | IAM OIDC identity provider + web identity role | Gives the autoscaler temporary credentials without static keys |
| Ingress to the app | Kubernetes `Service` of type `LoadBalancer` | Provisions an internet-facing Classic Load Balancer |
| Observability | CloudWatch, Prometheus metrics | Enabled by checkbox at cluster creation; not actively used |
| CLI | `aws`, `kubectl` | Cluster access and deployment |

---

## Quick start

Everything below was done in the AWS Management Console unless a command is shown.


### Prerequisites

- AWS account with permission to create IAM roles, VPC stacks and EKS clusters
- `aws` CLI configured (`aws configure list` shows the active region)
- `kubectl`

### Repository contents

```bash
git clone https://gitlab.com/cloudbuildops/eks-cluster-node-group.git
```

| File | Purpose |
|---|---|
| `cluster-autoscaler-autodiscovery.yaml` | ServiceAccount, RBAC (ClusterRole, Role and bindings) and Deployment for the Cluster Autoscaler |
| `custom-policy.json` | IAM policy attached to the autoscaler role |
| `nginx-config.yaml` | Sample nginx Deployment and `LoadBalancer` Service |

### 1. Create the EKS cluster IAM role

IAM is global, not regional.

1. IAM > Access Management > Roles > **Create role**
2. Trusted entity type: **AWS service**
3. Use case: **EKS**, then **EKS - Cluster** (not "EKS - Service")
4. Role name used here: `eks-cluster-role`

<img src="screenshots/01-iam-eks-cluster-role.png" alt="IAM role for the EKS cluster: AWS service, EKS - Cluster" width="800">

For this role type the console requires the AWS managed policy `AmazonEKSClusterPolicy`, and it is the only policy attached.

<img src="screenshots/01-iam-cluster-role-permissions.png" alt="Add permissions step: AmazonEKSClusterPolicy" width="800">

### 2. Create the VPC for the worker nodes

Use the AWS CloudFormation template for public and private subnets from the [EKS user guide](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html).

1. CloudFormation > **Create stack** > choose an existing template
2. Template source: Amazon S3 URL, using `https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml`
3. Stack name used here: `eks-worker-node-vpc-stack`
4. After creation, open the **Outputs** tab and note `SecurityGroups`, `SubnetIds` and `VpcId`. You need them for the cluster.

<img src="screenshots/02-cloudformation-create-stack.png" alt="CloudFormation create stack: existing template from an S3 URL" width="800">

The parameters are a `192.168.0.0/16` VPC with two public and two private `/18` subnets (full table under [Infrastructure](#infrastructure)).

<img src="screenshots/02-cloudformation-stack-parameters.png" alt="Stack details: name eks-worker-node-vpc-stack and CIDR parameters" width="800">

The stack ends in `CREATE_COMPLETE`, and its Outputs tab lists the 3 values the cluster needs.

<img src="screenshots/02-cloudformation-outputs.png" alt="CloudFormation Outputs: SecurityGroups, SubnetIds, VpcId" width="800">

