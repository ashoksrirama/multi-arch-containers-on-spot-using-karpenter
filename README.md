# Deploy Multi-Arch Containers on EC2 Spot using Karpenter and Amazon EKS
Sample Walkthrough on how to run multi-arch containers on EC2 Spot capacity using Karpenter. This setup will prioritize Graviton Spot capacity when available and fallback to x64 spot when unavailable.

## Pre-requisites
* [AWS Account](https://aws.amazon.com/free/)
* [AWS Command Line Interface](https://aws.amazon.com/cli/)
* [Terraform](https://developer.hashicorp.com/terraform/downloads)
* [eks-node-viewer](https://github.com/awslabs/eks-node-viewer)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [git](https://git-scm.com/downloads)

## Walkthrough

Clone the github repository

```shell
git clone https://github.com/ashoksrirama/multi-arch-containers-on-spot-using-karpenter
cd multi-arch-containers-on-spot-using-karpenter
```

Initialize and create plan for the Terraform code

```shell
terraform init
terraform plan
```

Run `Terraform Apply` to provision the the VPC, Subnets, Amazon EKS Cluster, Karpenter, and sample application

```shell
terraform apply -auto-approve
```

Update the kubectl context to interact with the Amazon EKS cluster

```shell
aws eks update-kubeconfig --name eks-multi-arch --region us-west-2
```

Verify the Karpenter installation by below command:

```shell
kubectl get pods -n karpenter
```
```output
NAME                         READY   STATUS    RESTARTS   AGE
karpenter-7c5df57794-m5gjm   1/1     Running   0          5m
karpenter-7c5df57794-z5d24   1/1     Running   0          5m
```

We are using a multi-architecture hello world application that returns the cpu architecture of the underlying Amazon EC2 worker node. It has a node affinity to prefer ARM64 nodes over x86 based instances.

```yaml
....
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values:
            - amd64
      - weight: 50
        preference:
          matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values:
            - arm64
....
```

```shell
kubectl get deploy
```
```output
NAME                READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS       IMAGES            SELECTOR
multi-arch-app      0/0     0            0           5m      multi-arch-app   sriram430/hello   app=multi-arch-app
```

A Karpenter `Provisioner` and `AWSNodeTemplate` resources are created to provision EC2 Spot instances.

```shell
kubectl get provisioners.karpenter.sh default -o yaml
```
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
  - key: "karpenter.k8s.aws/instance-category"
    operator: In
    values: ["c", "m", "r"]
....
  - key: "kubernetes.io/arch"
    operator: In
    values: ["arm64", "amd64"]
....
  - key: "karpenter.sh/capacity-type"
    operator: In
    values: ["spot"]
....
```

Lets test the setup by incrementing the # of replicas of multi-arch app

```shell
kubectl scale deploy multi-arch-app --replicas 50
```

Karpenter will start launching the EC2 instances as per the provisioner specs. You can monitor the EC2 instances by using `eks-node-viewer` tool

```shell
eks-node-viewer
```
```output
5 nodes (25850m/39850m) 64.9% cpu ██████████████████████████░░░░░░░░░░░░░░ $0.000/hour | $0.000/month
57 pods (3 pending 54 running 57 bound)

fargate-ip-10-0-38-105.us-west-2.compute.internal cpu ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  12% (1 pods)  0.25vCPU-0.5GB Fargate - Ready
fargate-ip-10-0-4-213.us-west-2.compute.internal  cpu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   0% (1 pods)  0.25vCPU-0.5GB Fargate - Ready
fargate-ip-10-0-44-10.us-west-2.compute.internal  cpu ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  12% (1 pods)  0.25vCPU-0.5GB Fargate - Ready
fargate-ip-10-0-38-126.us-west-2.compute.internal cpu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   0% (1 pods)  0.25vCPU-0.5GB Fargate - Ready
ip-10-0-27-132.us-west-2.compute.internal         cpu ████████████████████████████░░░░░░░  80% (53 pods) c6g.8xlarge    Spot    - Ready
```

You can notice Karpenter launched a c6g.8xlarge instance to accommodate 50 pending pods.

## Destroy

To avoid on-going costs, you can destroy the infrastructure using below command:

```shell
terraform destroy -auto-approve
```

## License

This library is licensed under the Apache License. See the [LICENSE](./LICENSE) file.
