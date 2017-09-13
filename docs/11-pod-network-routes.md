# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the VNet.

Print the internal IP address and Pod CIDR range for each worker instance, on each worker:

```
(hostname -I | cut -d' ' -f1 ) && echo $POD_CIDR
```

> output

```
10.240.0.20
10.200.0.0/24
```
```
10.240.0.21
10.200.1.0/24
```
```
10.240.0.22
10.200.2.0/24
```
## Routes

Create network routes for each worker instance:

```
az network route-table create -n $ROUTE_NAME -g $GROUP -l westeurope
az network route-table route create -n route0 -g $GROUP --route-table-name $ROUTE_NAME --address-prefix 10.200.0.0/24 --next-hop-type VirtualAppliance --next-hop-ip-address 10.240.0.20
az network route-table route create -n route1 -g $GROUP --route-table-name $ROUTE_NAME --address-prefix 10.200.1.0/24 --next-hop-type VirtualAppliance --next-hop-ip-address 10.240.0.21
az network route-table route create -n route2 -g $GROUP --route-table-name $ROUTE_NAME --address-prefix 10.200.2.0/24 --next-hop-type VirtualAppliance --next-hop-ip-address 10.240.0.22
az network vnet subnet update -n $SUBNET_NAME -g $GROUP --vnet-name $VNET_NAME --route-table $ROUTE_NAME
```


Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
