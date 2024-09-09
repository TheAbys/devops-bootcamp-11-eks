# 1 - COntainer Services on AWS

There are multiple solutions to handle containers in the cloud, self-managed, semi-managed and fully-managed.

# 2 - Create EKS cluster with AWS Management Console

We've created the complete EKS cluster and all dependencies within AWS.
Through CloudFormation and a public template the necessary VPC and other network configuration was prepared for EKS.
The default VPCs are not optimized for EKS.

The cluster API Server must be configured public and private so that the API-Server is accessable from outside the VPC but also directly from inside.
Otherwise kubectl can't be used on my local machine.

Create IAM role for EKS Cluster

    k0938261_training_eks-cluster-role

Create VPC because default VPC is not EKS optimized
As the Control plane nodes are AWS managed they are connected in another VPC and must be able to communicate with the worker nodes in our VPC.

VPCs are created through CloudFormation
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml

But this doesn't work because of company restrictions or because the subnets are already in use.

After the EKS Cluster is set up:

    aws eks update-kubeconfig --name eks-cluster-test

Show API-Server url

    kubectl cluster-info

## Setup kube-config locally

    aws eks update-kubeconfig --name eks-cluster-test

updates the /home/mueller/.kube/config file and adds the context if not already configured

When setting up the cluster only the Control Plane Nodes are created. Worker nodes must be created afterwards.


# 3 - Configure Autoscaling in EKS cluster

Autoscaling is not happening automatically by default. It must be configured through multiple steps.
A auto scaling group for ec2 instances is automatically created, but this group does not automatically scale everything.
A policy is necessary especially for auto scaling which must be attached to the previously created eks-node-group role.

Afterwards we need to deploy the autoscaler into our kubernetes cluster.

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

Update the deployment for a few changes, added multiple options and adjusted the autoscaler version to match the kubernetes version (1.29 in our case)

    kubectl edit deployment -n kube-system cluster-autoscaler

    kubectl get pod -n kube-system

for each worker node a kube-proxy and aws-node pod is created
coredns as always to replica

    kubectl get pod -n kube-system -o wide

to check on which worker node the autoscaler pod was deployed


    kubectl logs -n kube-system -f cluster-autoscaler-5d78f65894-tp2kj

it takes some time but when the autoscaler runs again and it sees changes within the desired, minimum and maxium configuration it automatically up or downscales.
Benefit of this is the cost reducation if there is no load. It takes some time to spin up a new worker node though.


    kubectl apply -f nginx-config.yaml

