# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single availability zone. Pleae note htat you may very well decide to build the control plane and the worker nodes in multiple AZs. 

We have chosen AWS US-WEST-2, in only us-west-2b Availabilty zone. you're welcome to choose any region. 


> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-aws-way` custom VPC network:

```
aws ec2 create-vpc  --cidr-block 10.240.0.0/16  
```

A [subnet](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `kubernetes-the-aws-way` VPC network:

```
aws ec2 create-subnet --vpc-id vpc-2f09a348 --cidr-block 10.0.1.0/24

```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Create an Internet Gateway

```
aws ec2 create-internet-gateway
```
### Attached the Internet Gateway to the VPC
```
aws ec2 attach-internet-gateway --vpc-id $vpc_id --internet-gateway-id $internet_gateway
```
### Create Route Table

```
aws ec2 create-route-table --vpc-id $vpc_id
```

### Create a route in the route table that points all traffic (0.0.0.0/0) to the internet gateway
```
aws ec2 create-route --route-table-id $route_table_id --destination-cidr-block 0.0.0.0/0 --gateway-id $internet_gateway
```

Confirm the route table has a route to internet:
```aws ec2 describe-route-tables --route-table-id $route_table_id```
Associate the route table to the subnet 
```aws ec2 associate-route-table --subnet-id $subnet_id --route-table-id $route_table_id```

### Create firewall Rules (Security Groups)
Create a security group in your VPC, and add a rule that allows SSH access from anywhere.
```aws ec2 create-security-group --group-name SSHAccess --description "Security group for SSH access" --vpc-id $vpc_id```
Security group for internal access

```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=icmp,FromPort=0,ToPort=4,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]’ ```

```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=tcp,FromPort=0,ToPort=65000,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]’```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=udp,FromPort=0,ToPort=65000,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]' ```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp  --cidr 10.240.0.0/24 ```
Security group for external access 
 ```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=icmp,FromPort=0,ToPort=4,IpRanges='[{CidrIp=0.0.0.0/0}]'```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp --port 22 --cidr 0.0.0.0/0 ```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp --port 6443 --cidr 0.0.0.0/0 ```

Create a static Public IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

``` aws ec2 allocate-address ```
Record the allocation id into `$allocation_id` and tag it as shown below

``` aws ec2 create-tags --resource $allocation_id --tags Key=Name,Value=kubernetes-the-hard-way ```

### worker node elastic IP addresses

### worker node Elastic Network Interfaces 

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-aws-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-aws-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-gcp x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
