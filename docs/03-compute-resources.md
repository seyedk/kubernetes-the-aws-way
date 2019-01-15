# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single availability zone. Pleae note htat you may very well decide to build the control plane and the worker nodes in multiple AZs. 

We have chosen AWS US-WEST-2, in only us-west-2b Availabilty zone. you're welcome to choose any region. 


> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Env. Variables
let's create a file to keep the env. variables that are set in each step so we can pick up from where we left off 

```
 {
   touch kaw.sh
   chmod 744 kaw.sh
  
 }
```

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) (VPC) network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-aws-way` custom VPC network:

```aws ec2 create-vpc  --cidr-block 10.240.0.0/16```

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

```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=icmp,FromPort=0,ToPort=4,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]' ```

```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=tcp,FromPort=0,ToPort=65000,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]'```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=udp,FromPort=0,ToPort=65000,IpRanges='[{CidrIp=10.240.0.0/24}, {CidrIp=10.200.0.0/16}]' ```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp  --cidr 10.240.0.0/24 ```

Security group for external access 
 ```aws ec2 authorize-security-group-ingress --group-id $sg_group_id --ip-permissions IpProtocol=icmp,FromPort=0,ToPort=4,IpRanges='[{CidrIp=0.0.0.0/0}]'```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp --port 22 --cidr 0.0.0.0/0 ```

``` aws ec2 authorize-security-group-ingress --group-id $sg_group_id --protocol tcp --port 6443 --cidr 0.0.0.0/0 ```

Create a static Public IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```aws ec2 allocate-address```

Record the allocation id into `$allocation_id` and tag it as shown below

``` aws ec2 create-tags --resource $allocation_id --tags Key=Name,Value=kubernetes-the-aws-way ```

### worker node elastic IP addresses
In this step you will need to create elastic IP addresses that can be later associated with Network Interfaces
```
{
  for i in 0 1 2 ; do
    eip = $(aws ec2 allocate-address --profile beta-lab | jq -r .AllocationId)
    aws ec2 create-tags --resource $eip --tags Key=Name,Value=worker-${i}
    echo export Worker${i}Alloc=${eip} >> kaw.sh
  done
  source ./kaw.sh
}
  
```

### worker node Elastic Network Interfaces 
Now let's create the ENIs that will be attached to the worker instances during the launch time. 
```
{
    for i in 0 1 2 ; do
    eniId=$(aws ec2 create-network-interface \
            --description 'Primary Network interface for worker-${i}' \
            --no-dry-run \
            --groups $securityGroupId \
            --private-ip-address 10.240.0.2${i} \
            --subnet-id $subnetId | jq -r '.NetworkInterface.NetworkInterfaceId' \
            )
    
    aws ec2 create-tags --resource $eniId --tags Key=Name,Value=worker-${i};
    echo export worker${i}Eni=$eniId >> kaw.sh
    done

  
  source ./kaw.sh
}

```

### Assoicate the Elastic IP Addresses 
in this step we should assoicate the ip addresses of the Worker nodes to the Wroker Nodes Elastic Network Interfaces 

```
for i in 0 1 2; do
  associate-address \
    --allocation-id Worker${i}Alloc \
    --network-interface-id $worker${i}Eni 
done

```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
allocation_id=$(aws ec2 allocate-address | jq -r '.AllocationId')
```


 Then you need to tag it as shown below
```aws ec2 create-tags --resource $allocation_id --tags Key=Name,Value=kubernetes-the-aws-way```


> PLEASE MAKE SURE VERIFY THE ELASTIC IP ADDRESS HAS BEEN CREATED BEFORE ATTACHIGN TO KUBERNETES ALB.


## Compute Instances

### Create a private key for SSH access using AWS EC2 

The Keypair will be used for all ec2 instances.

```
{
  keypairName=seyedkey;
  aws ec2 create-key-pair --key-name $keypairName | jq -r '.KeyMaterial' > ${keypairName}.pem

  chmod 600 $keypairName.pem

  ssh-add -k $keypairName.pem

  echo export keypairName=$keypairName >> kaw.sh ;
  source kaw.sh;
}
```

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
echo ${i};
aws ec2 run-instances \
 --image-id ami-0bbe6b35405ecebdb \
 --count 1 \
 --instance-type t2.micro \
 --key-name $keypairname \
 --security-group-ids $secGroupId \
 --subnet-id $subnetId \
 --private-ip-address 10.240.0.1${i} \
--associate-public-ip-address \
--disable-api-termination \
 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=controller-$i},{Key=hol,Value=kaw}]"
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance `user-data` will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 1 2 ; do

n=$(aws ec2 describe-network-interfaces --filter "Name=tag:Name,Values=worker-${i}" --query NetworkInterfaces[0].NetworkInterfaceId | tr -d '"');

echo ${n};

aws ec2 run-instances \
--no-dry-run \
--image-id ami-0bbe6b35405ecebdb \
--count 1 \

--instance-type t2.micro \
--key-name $keypairname \
--disable-api-termination \
--user-data pod-cidr=10.200.${i}.0/24 \
--network-interfaces "{\"DeviceIndex\":0,\"NetworkInterfaceId\": \"${n}\"}" \
--tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=worker-${i}},{Key=hol,Value=khw}]"

done
```

### Verification

List the compute instances in your default compute zone:

```
aws ec2 describe-instances | jq -r 'Reservations[].Instances[]'
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

SSH will be used to configure the controller and worker instances. In order to proceed to the next configuration steps, we add the IP addresses of the instances into the /etc/hosts file. this step is left open for contribution. 

```
  pleae provide the code that gets the public address of each instance, and based on its tag, adds an entry to /hosts file
```



Test SSH access to the `controller-0` compute instances:

```
 ssh ubuntu@controller-0
```
 you'll be logged into the `controller-0` instance:

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
