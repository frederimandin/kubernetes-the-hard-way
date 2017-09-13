# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single resource group, that will allow for simple cleanup and control.

> Ensure the default variables have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Networking resources

In this section a dedicated [Virtual Network](https://azure.microsoft.com/en-us/services/virtual-network/) (VNet) network will be setup to host the Kubernetes cluster.

Create the custom VNet network:

```
az network vnet create -n $VNET_NAME -g $GROUP --address-prefixes 10.240.0.0/16 10.200.0.0/16
```

As we will want to secure the resources, a Network Security Group is to be created :
```
az network nsg create -n $NSG_NAME -g $GROUP -l westeurope
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-internal --priority 100 --access Allow --source-address-prefix 10.240.0.0/24
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-internal2 --priority 101 --access Allow --source-address-prefix 10.200.0.0/16
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-external3 --priority 102 --access Allow --destination-port-range 22
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-external --priority 103 --access Allow --destination-port-range 3389
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-external2 --priority 104 --access Allow --destination-port-range 6443
az network nsg rule create -g $GROUP --nsg-name $NSG_NAME -n allow-healthz --priority 105 --access Allow --destination-port-range 8080
```

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the subnet in the VNet:

```
az network vnet subnet create -g $GROUP --vnet-name $VNET_NAME -n $SUBNET_NAME --address-prefix 10.240.0.0/24 --network-security-group $NSG_NAME
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Kubernetes Public IP Addresses

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
az network public-ip create -n $PUBLIC_IP_NAME -l westeurope -g $GROUP  --allocation-method static
KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g $GROUP -n $PUBLIC_IP_NAME --query "{ address: ipAddress }" -o tsv)
```

Create 3 public IPs for the 3 controller VMs:

```
az network public-ip create -n controller0IP -l westeurope -g $GROUP  --allocation-method static
az network public-ip create -n controller1IP -l westeurope -g $GROUP  --allocation-method static
az network public-ip create -n controller2IP -l westeurope -g $GROUP  --allocation-method static
```

### Load Balancer
We will need a load balancer to redirect the requestes between all three controllers. This LB will need the 3 VMs to be in the same availability set, that we will create also.

Create the availability set:
```
az vm availability-set create -n $ASET_NAME -g $GROUP -l westeurope
```

Create the API frontend load balancer, frontend IP, probe and load balancing rule
```
az network lb create -g $GROUP -n $LOAD_BALANCER -l westeurope --backend-pool-name $POOL_NAME --public-ip-address $PUBLIC_IP_NAME  --public-ip-address-type Public 
az network lb probe create --lb-name $LOAD_BALANCER -n $PROBE_NAME --port 8080 --protocol http -g $GROUP --path /healthz
az network lb rule create --backend-port 6443 --frontend-port 6443 --lb-name $LOAD_BALANCER -n $RULE_NAME --protocol tcp -g $GROUP --backend-pool-name $POOL_NAME --probe-name $PROBE_NAME
```

Create the NICs attached to the load balancer pools
```
az network nic create -g $GROUP -n controller0VMNic --vnet-name $VNET_NAME --subnet $SUBNET_NAME --lb-name $LOAD_BALANCER --lb-address-pools $POOL_NAME --private-ip-address 10.240.0.10 --public-ip-address controller0IP
az network nic create -g $GROUP -n controller1VMNic --vnet-name $VNET_NAME --subnet $SUBNET_NAME --lb-name $LOAD_BALANCER --lb-address-pools $POOL_NAME --private-ip-address 10.240.0.11 --public-ip-address controller1IP
az network nic create -g $GROUP -n controller2VMNic --vnet-name $VNET_NAME --subnet $SUBNET_NAME --lb-name $LOAD_BALANCER --lb-address-pools $POOL_NAME --private-ip-address 10.240.0.12 --public-ip-address controller2IP
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) latest supported version, which has good support for the [CRI-O container runtime](https://github.com/kubernetes-incubator/cri-o). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
az vm create -n controller-0 -g $GROUP --image UbuntuLTS --availability-set $ASET_NAME --nics controller0VMNic --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
az vm create -n controller-1 -g $GROUP --image UbuntuLTS --availability-set $ASET_NAME --nics controller1VMNic --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
az vm create -n controller-2 -g $GROUP --image UbuntuLTS --availability-set $ASET_NAME --nics controller2VMNic --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. 

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.


Create three compute instances which will host the Kubernetes worker nodes:

```
az vm create -n worker-0 -g $GROUP --image UbuntuLTS --vnet-name $VNET_NAME --subnet $SUBNET_NAME --private-ip-address 10.240.0.20 --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
az vm create -n worker-1 -g $GROUP --image UbuntuLTS --vnet-name $VNET_NAME --subnet $SUBNET_NAME --private-ip-address 10.240.0.21 --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
az vm create -n worker-2 -g $GROUP --image UbuntuLTS --vnet-name $VNET_NAME --subnet $SUBNET_NAME --private-ip-address 10.240.0.22 --authentication-type password --admin-username $SSH_USERNAME --admin-password $SSH_PASSWORD --nsg ""
```

### Verification

List the compute instances in your default compute zone:

```
az vm list -g $GROUP -d -o table
```

> output

```
Name         ResourceGroup    PowerState    PublicIps      Location
-----------  ---------------  ------------  -------------  ----------
controller-0  GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
controller-1  GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
controller-2  GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
worker-0      GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
worker-1      GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
worker-2      GroupName     VM running    XXX.XXX.XXX.XXX    westeurope
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
