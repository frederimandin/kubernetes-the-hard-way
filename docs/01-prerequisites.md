# Prerequisites

## Microsoft Azure

This tutorial leverages [Microsoft Azure](https://portal.azure.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://azure.microsoft.com/fr-fr/free/) for $200 in free credits.

## Azure CLI

### Install Azure CLI

Follow the Azure CLI [documentation](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest) to install and configure the `az` command line utility.
Azure CLI is working:

```
az
```
Output 

```

     /\
    /  \    _____   _ _ __ ___
   / /\ \  |_  / | | | \'__/ _ \
  / ____ \  / /| |_| | | |  __/
 /_/    \_\/___|\__,_|_|  \___|


Welcome to the cool new Azure CLI!

Here are the base commands:

    account          : Manage subscriptions.
    acr              : Manage Azure container registries.
    acs              : Manage Azure Container Services.
    ad               : Synchronize on-premises directories and manage Azure Active Directory
                       resources.
...
```

### Set the overall variables you will need

This tutorial will need several names already setup, for all resources :

```
AZURE_SUBSCRIPTION=Id_of_the_correct_Azure_subscription
SSH_PASSWORD=Password_for_SSH_connection_to_the_hosts
SSH_USERNAME=Username_to_use_for_SSH_connection
GROUP=Name_of_the_resource_group
VNET_NAME=Name_of_the_Vnet
NSG_NAME=Name_of_the_NSG
SUBNET_NAME=Name_fo_the_Subnet
PUBLIC_IP_NAME=Name_of_the_Public_IP
LOAD_BALANCER=Name_of_the_lb
POOL_NAME=Name_of_the_LB_pool
RULE_NAME=Name_fo_the_load_balancer_rule
PROBE_NAME=Name_of_the_probe
ASET_NAME=Name_of_the_availability_set
ROUTE_NAME=Name_of_the_route_table
```

Then we will need to set the correct subscription to be used :
```
az account set -s $AZURE_SUBSCRIPTION
```

And create the resource group that will regroup all the necessary resources
```
az group create -l westeurope -n $GROUP
```

Next: [Installing the Client Tools](02-client-tools.md)
