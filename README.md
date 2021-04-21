
# Setup EKS Managed Node Groups with Bottlerocket using Launch Templates

## Introduction

This walkthrough is intended to get you up and running with a [Managed Node Group](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) using Bottlerocket (https://aws.amazon.com/bottlerocket/) nodes, and to familiarize you with some of the differences between a managed node group using the default Amazon Linux 2 nodes and a managed node group using Bottlerocket nodes.

**NOTE: In the future, managed node groups will support Bottlerocket as a built-in OS choice.** Until then, this walkthrough should provide a reliable set of steps to build a managed node group with Bottlerocket nodes using [launch templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html).

## Prerequisites

* *Basic familiarity with EKS.* For this walkthrough, we will assume some familiarity with EKS and its basic concepts, and that you have started EKS clusters successfully and have run basic workloads on them.
* *Command-line tools.* On your local workstation, you will need to install the latest versions of [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html), [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html), and the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).
* *Permissions.* For this walkthrough, you should ensure that the IAM user that you use to create the eksctl cluster also has access to the AWS Management console, including [access to the EKS console](https://docs.aws.amazon.com/eks/latest/userguide/security_iam_id-based-policy-examples.html#security_iam_id-based-policy-examples-console). 

## Step 1: Create the Cluster and Verify Console Access

You’ll start by creating a simple example cluster using eksctl. It’s possible to use the AWS management console to create your cluster, but using eksctl will give you a set of simple defaults, and will also allow easy command line access later.

```
eksctl create cluster --name [your-cluster-name] --region [your-region] --without-nodegroup
```

In our example, we will call our cluster “purple-goose” and build it in region us-east-2. By default, when eksctl creates a new cluster, it also creates an unmanaged node group with 2 nodes. Here we specify the extra argument `--without-nodegroup` so that you don’t have to delete those unused nodes later.

After creating your cluster with eksctl, we will ask you to move to the AWS management console for the remainder of the walkthrough, for a couple of reasons. First, it’s easier to create launch templates in the console than by using command line tools, and second, the interactive nature of the console interface allows users to catch errors that they might miss using command line tools. Once you understand the basic steps, you can then use various AWS CLI tools to perform the same actions.

Creating the initial cluster can take time. Once the cluster is created, sign into the console, select the “Elastic Kubernetes Service”, and under “Amazon EKS” on the left nav, click on “Clusters”. You should see a list of clusters;  verify that the cluster you just created exists and is in the “Active” state.

<img width="1320" alt="mng-001" src="https://user-images.githubusercontent.com/252125/115579970-8589a680-a27b-11eb-979e-02d2c8d72ddf.png">

Note that at the time of writing this,  eksctl creates a cluster with a default Kubernetes version of 1.18; because version 1.19 is already available, you see a link to update the new cluster. Stay with your 1.18 cluster for now.

Now click on the cluster name. If you see details of the cluster, you should be able to move on to Step 2.

If you receive an error that your console user doesn’t have access to the cluster, that means that you created the cluster with a different IAM Role or User ARN than you’re using to access the console.  For an example of how to align IAM credentials properly betweeen eksctl and your console, see the Console Credentials section of the [Amazon EKS Workshop](https://www.eksworkshop.com/030_eksctl/console/).

Now it’s time to gather information about your cluster for use in later steps.

## Step 2: Gather Information About the Cluster

Here’s a list of the information you’ll need to gather, why you need it, and where to get it. Be sure to note these values for later use.

* *Bottlerocket AMI ID.* The simplest and most accurate way to get the latest Bottlerocket AMI ID for Kubernetes 1.18 is to run the following command on your local workstation: 

```
aws ssm get-parameter --region [your-region] \ 
    --name "/aws/service/bottlerocket/aws-k8s-1.18/[x86_64|arm64]/latest/image_id" \
    --query Parameter.Value --output text
```

* *Cluster VPC ID.* You will need this later when to add security rules. In the console, go to *EKS > Clusters > [your cluster name] > Configuration > Networking*. You will find the VPC of your cluster here; record the VPC ID for later use. 

* *API Server Endpoint and Certificate Authority.* You will need to put these into the userdata section of the launch template. Bottlerocket nodes need this userdata so that they can communicate with the cluster API; if this data is not present, the Bottlerocket nodes will launch but fail to connect to the cluster, and your transaction will fail. Go to *EKS > Cluster > [your cluster name] > Configuration > Details*. Your console screen should look similar to the screenshot below; copy the API Server Endpoint and the Certificate Authority and save those values for later use.

<img width="1304" alt="mng-003" src="https://user-images.githubusercontent.com/252125/115580223-bff34380-a27b-11eb-96d0-106fd372e957.png">

## Step 3: Create an IAM Role for Your Bottlerocket Managed Node Group

This role is necessary for the kubelet daemon on your Bottlerocket nodes to be able to talk to the cluster; if it is not properly configured, the nodes will not connect to the cluster, and your node group will fail to launch.  *Follow the instructions for [Creating the Amazon EKS IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role)*, but where the instructions tell you to add two policies, AmazonEKSWorkerNodePolicy and AmazonEC2ContainerRegistryReadOnly, you should also add two additional policies: AmazonSSMManagedInstanceCore, so that Bottlerocket’s control container can communicate with SSM, and AmazonEKS_CNI_Policy. Click through to the Review step, where your final role should look like this:

<img width="991" alt="mng-004" src="https://user-images.githubusercontent.com/252125/115580279-cc779c00-a27b-11eb-995a-1eac8a19a5cd.png">

Click the “Create” button and your new role will be created. 

## Step 4: Create a security group to allow SSH access to the admin container

Using eksctl to create a cluster will create all of the necessary security groups for internal communication, but it will not create a security group allowing for ssh access. Ordinarily, the admin container is disabled so that you can’t ssh to the nodes at all, but when you want to enable ssh in the admin container, you want to be sure that you can actually connect to it.

To create a security group, go to *EC2 > Security Groups* and click the “Create Security Group” button. Provide a name for the security group; for the VPC pulldown, select the VPC that you recorded in Step 2. Be sure to select the right VPC; if the security group is linked to the wrong VPC, it won’t be available to you later. Next, under Inbound Rules, add a rule, type “SSH”, source “My IP”. Your new security group should look something like this:

<img width="1568" alt="mng-new-sg-2" src="https://user-images.githubusercontent.com/252125/115580330-d4cfd700-a27b-11eb-8bae-a24d0e122a7e.png">

Save your new security group.

## Step 5: Create Your Launch Template

EKS added support for [EC2 Launch Templates](https://aws.amazon.com/blogs/containers/introducing-launch-template-and-custom-ami-support-in-amazon-eks-managed-node-groups/) to support custom AMIs and managed node groups, and you will use this feature to configure your template for Bottlerocket nodes. 

To set up your Launch Template, go to *EC2 > Launch Templates > Create Launch Template*.

There are several fields that you might choose to set, but for this walkthrough, only change the following fields, which are the minimum fields required for a successful launch of Bottlerocket nodes. Leave other fields unchanged for now. Once you have a reliable template, you can then make versioned changes to your launch template to try different parameters. 

* *Launch Template Name.* Give your launch template a name. We will call our example “purple-goose-lt”.
* *AMI.* Specify a custom value, and provide the AMI ID you selected in Step 2.
* *Instance Type.* Pick a supported instance type. Bottlerocket builds from AWS are supported on HVM and EC2 Bare Metal instance families with the exception of the P, G, F, and INF instance types. In our example, we have chosen m5.large. 
* *Key pair.* Choose a key pair. The only way to ssh into a Bottlerocket node is to ssh into its admin container; you will enabling the admin container in the user data, below. The key pair you select here will be made available to the admin container when it is instantiated. 
* *Networking setting.* Leave as VPC.
* *Subnet.* Don’t include any subnet information in the template. When you go to launch the node group later, you will choose an appropriate subnet from data that will be auto-populated from the cluster information.
* *Security Groups.* You will select two security groups here: the ssh security group that you created in Step 4, and the Cluster Security Group. When eksctl creates a cluster, it creates several security groups. One of these is the Cluster Security Group, which is a unified security group that allows communication between all components. You can easily identify this security group because it begins with eks-cluster-sg-[your-cluster-name]. Choose “select existing security groups”, then from the pulldown menu, choose the Cluster Security Group, and the ssh security group that you created in Step 4.
* *Advanced details / User data.* This is critically important. If any data in this field is absent or malformed, the nodes will not be able to connect to the cluster. Below is a sample user data, which provides the necessary settings to Kubernetes and also enables the admin container on the Bottlerocket hosts. Use the following format below, and replace the cluster name, API server, and cluster certificate fields with the the data you collected in Step 2. Note also that we have enabled the admin container to test our ability to ssh to the nodes; in future revisions of the launch template, you should remove this option, since the admin container should only be running for debugging or troubleshooting situations. Cut and paste the resulting user data settings into the “Advanced Settings / User data” field at the bottom of the launch template.

```
[settings.kubernetes]
cluster-name = "[your-cluster-name]"
api-server = "[api-server-url]"
cluster-certificate = "[full certificate here]"
[settings.host-containers.admin]
enabled = true
```

Save your launch template.

## Step 6: Launch Your Managed Node Group

Now you should have all of the pieces in place to launch Bottlerocket instances into a managed node group. 

Go to *EKS > Clusters > [your cluster name] > Configuration > Compute*, where you will see a button labeled “Add Node Group”. Click that button. Note that this will launch a managed node group; in this interface, only managed node groups can be launched. 

On the next page, give your new managed node group a name. For “Node IAM Role” select the role you created in Step 3, and under “Launch Template” select “Use Launch Template”. Under “Launch Template Name” select the launch template you created in Step 4. For version, select “Version 1”. Your configuration should look something like this:

<img width="864" alt="mng-005" src="https://user-images.githubusercontent.com/252125/115580383-e1ecc600-a27b-11eb-8c68-4df927f49e46.png">

Then click “Next”, where you will be able to choose the minimum, maximum, and desired size of your node group; a default of 2 is fine for this walkthrough. You can leave all other values at their defaults.

Click “Next” again, where you will see that a number of subnets have been automatically created. For our example, we want to choose a public subnet, so that we can ssh in to test the admin container. Select one of the subnets from the pulldown menu that has “public” in its name, and unselect the others.

Click “Next” again. You will be shown the parameters for final review, and then click “Create”, which will start the node group creation process:

<img width="1336" alt="mng-006" src="https://user-images.githubusercontent.com/252125/115580430-ec0ec480-a27b-11eb-8f67-295ed0d43049.png">

Assuming you followed all the steps, you should see your managed node group within a few minutes, with its status now set to “Active”:

<img width="1292" alt="mng-007" src="https://user-images.githubusercontent.com/252125/115580465-f29d3c00-a27b-11eb-81c5-3bd9969a0401.png">

Your managed node group is now available to accept workloads. To ensure that your managed node group is running as expected, you can run the following kubectl command from your local workstation:

```
kubectl get nodes -o wide
```

You should see that your Bottlerocket nodes are up and running, and you should see public IP addresses for your nodes. 

Now it’s time to test your ssh access to the admin container. Select one of your nodes and run the following command:

```
ssh -i /path/to/your-ssh-key ec2user@[node-public-ip-address]
```

You should see the admin container’s welcome message, with further instructions for how to use the admin container. Before you go into production, you should disable the admin container; simply create a new revision to the launch template and remove the admin container parameter from the user data section. The admin container can always be reenabled by connecting to the Bottlerocket control container via SSM; see the [documentation](https://github.com/bottlerocket-os/bottlerocket/blob/develop/README.md#control-container) for more details.

## Next Steps

Now that you have your managed node group running, you can more fully explore the functionality of managed node groups with your Bottlerocket nodes. 

If you have questions, please feel free to join us in the [Bottlerocket discussion forum](https://github.com/bottlerocket-os/bottlerocket/discussions), where the Bottlerocket team will be happy to answer any questions you may have. 
