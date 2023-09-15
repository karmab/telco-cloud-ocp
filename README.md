This repository provides automation for deploying:

- Two Openshift UPI private clusters in two different Availability Zones (AZ) and with multinics workers
- Those clusters are running in dedicated private subnets
- The resulting clusters contain cloud storage and cloud loadbalancing

The primary cloud environment is AWS although most of the steps are expected to work similarly in other supported platforms (GCP and Azure in particular)

# Requirements

## Aws account

You will need to gather the following elements from your AWS account:

- `access_key_id`
- `access_key_secret`
- The DNS domain to use 

We will use the regions eu-west-3 and us-east-2, but any pair of regions would be fine.

You can also decide too work on a single region

## Kcli

We leverage [kcli](https://kcli.readthedocs.io/en/latest/) since it has support for creating vms and clusters in cloud platforms

### Installation

We can install it using the following one liner (that will either use package based or container approach for deploying)

```
curl https://raw.githubusercontent.com/karmab/kcli/main/install.sh | sudo bash
```

Running on aws requires to install additional dependencies (boto3), which can be done with

```
kcli install provider aws
```

### Configuration

Once kcli is installed, we configure an aws provider in ~/.kcli/config.yml by creating the following section

```
default:
 client: aws
aws:
  type: aws
  access_key_id: CHANGEME
  access_key_secret: CHANGEME
  enabled: true
  region: eu-west-3
```

## Deployment

### Vpcs and Subnets

We deploy all the subnets needed in the eu-west-3 region by running the following commands. 

```
VPC="myvpc-eu"
kcli -z eu-west-3a create network -P cidr=11.0.0.0/16 -P subnet_cidr=11.0.1.0/24 $VPC
kcli -z eu-west-3a create subnet --cidr 11.0.2.0/24 -P network=$VPC -i $VPC-subnet2

kcli -z eu-west-3b create subnet --cidr 11.0.3.0/24 -P network=$VPC $VPC-subnet3
kcli -z eu-west-3b create subnet --cidr 11.0.4.0/24 -P network=$VPC -i $VPC-subnet4

kcli -z eu-west-3c create subnet --cidr 11.0.5.0/24 -P network=$VPC $VPC-subnet5
kcli -z eu-west-3c create subnet --cidr 11.0.6.0/24 -P network=$VPC -i $VPC-subnet6
```

This also creates under the hood:

- a dedicated vpc
- an internet gateway attached to each odd subnet
- a nat gateway attached to each even subnet


To do the same in the us-east-2 region, we run the following commands:

```
VPC="myvpc-us"
kcli -r us-east-2 -z us-east-2a create network -P cidr=12.0.0.0/16 -P subnet_cidr=12.0.1.0/24 $VPC
kcli -r us-east-2 -z us-east-2a create subnet --cidr 12.0.2.0/24 -P network=$VPC -i $VPC-subnet2

kcli -r us-east-2 -z us-east-2b create subnet --cidr 12.0.3.0/24 -P network=$VPC $VPC-subnet3
kcli -r us-east-2 -z us-east-2b create subnet --cidr 12.0.4.0/24 -P network=$VPC -i $VPC-subnet4

kcli -r us-east-2 -z us-east-2c create subnet --cidr 12.0.5.0/24 -P network=$VPC $VPC-subnet5
kcli -r us-east-2 -z us-east-2c create subnet --cidr 12.0.6.0/24 -P network=$VPC -i $VPC-subnet6
```

Optionally, we can link the two vpcs together via a peering connection by running the following python script

```
from kvirt.config import Kconfig
from time import sleep

network1, network2 = 'myvpc-eu', 'myvpc-us'
region1 = 'eu-west-3'
region2 = 'us-east-2'

config1 = Kconfig(region=region1)
config2 = Kconfig(region=region2)

vpc_network1 = config1.k.get_vpc_id(network1) if not network1.startswith('vpc-') else network1
vpc_network2 = config2.k.get_vpc_id(network2) if not network2.startswith('vpc-') else network2

response = config1.k.conn.create_vpc_peering_connection(VpcId=vpc_network1, PeerVpcId=vpc_network2, PeerRegion=region2)
peering_id = response['VpcPeeringConnection']['VpcPeeringConnectionId']
sleep(20)
config2.k.conn.accept_vpc_peering_connection(VpcPeeringConnectionId=peering_id)
```

### Bastion instances

We will create a bastion instance in each vpc by collocating it in the corresponding public subnet.
Those instances will run sshuttle to allow us to reach the private networks (and as such require python3 to be installed)


```
VPC="myvpc-eu"
kcli -z eu-west-3a create vm -i "CentOS Stream 8 x86_64" -P nets=[$VPC-subnet1] -P cmds=['yum -y install python3'] bastion-$VPC
```

```
VPC="myvpc-us"
kcli -r us-east-2 -z us-east-2a create vm -i "CentOS Stream 8 x86_64 20230710" -P nets=[$VPC-subnet1] -P cmds=['yum -y install python3'] bastion-$VPC
```

### Openshift deployment

#### Sshuttle

We use sshuttle to give access from our laptop to the public subnets

Alternatively, we can install kcli in the bastion node and launch deployment from there


To access the entire myvpc-eu network from the outside, we run the following

```
VPC="myvpc-eu"
IP=$(kcli info vm -vf ip bastion-$VPC)
sshuttle -r root@$IP 11.0.0.0/16
```

Similarly, to access the entire myvpc-us network from the outside, we run the following

```
VPC="myvpc-us"
IP=$(kcli info vm -vf ip bastion-$VPC)
sshuttle -r root@$IP 12.0.0.0/16
```


We deploy our eu-west based Openshift cluster by running the following command

```
kcli create cluster openshift --pf parameters_eu.yml cluster-eu-0
```

The us-east cluster can be deployed with

```
kcli create cluster openshift --pf parameters_us.yml cluster-us-0
```


## Sample App

We will deploy a sample `reversewords` app in the first cluster 

```
oc create -f sampleapp/deploy.yml
oc create -f sampleapp/service.yml
```

Note that we can make the corresponding LB ip private by using the commented annotation, but the service wont be available from myvpc-us then.

## Connectivity between vpc and onprem network

We solve this using tailscale

On the bastion instance, we run

```
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=11.0.0.0/16 --accept-routes
```

On one of our onprem box (in this case a libvirt hypervisor), we run 

```
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=192.168.122.0/24 --accept-routes
```

We then accept the subnet routes in the tailscale admin console and from then, we have connectivity between the two subnets

We also need to add a route in the route table of the AWS private subnet so that they reach the onprem network via our bastion instance




