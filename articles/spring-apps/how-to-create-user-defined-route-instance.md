---
title:  Control egress traffic for an Azure Spring Apps instance
description: Learn how to control egress traffic for an Azure Spring Apps instance.
author: karlerickson
ms.author: yinglzh
ms.service: spring-apps
ms.topic: article
ms.date: 09/25/2021
ms.custom: devx-track-java, devx-track-azurecli
---

# Control egress traffic for an Azure Spring Apps instance

**This article applies to:** ✔️ Java ✔️ C#

**This article applies to:** ✔️ Basic/Standard tier ✔️ Enterprise tier

This article describes how to secure outbound traffic from your applications hosted in Azure Spring Apps. The article provides an example of a user-defined route (UDR) instance. UDR is an advanced feature that lets you fully control egress traffic. You can use UDR in scenarios such as disallowing an Azure Spring Apps auto-generated public IP.

## Prerequisites

- All prerequisites for deploying Azure Spring Apps in a virtual network. For more information, see [Deploy Azure Spring Apps in a virtual network](how-to-deploy-in-azure-virtual-network.md).
- API version of `2022-09-01 preview` or greater
- [Azure CLI version 1.1.7 or later](/cli/azure/install-azure-cli).
- You should be familiar with information in the following articles:
  - [Deploy Azure Spring Apps in a virtual network](how-to-deploy-in-azure-virtual-network.md)
  - [Customer responsibilities for running Azure Spring Apps in VNET](vnet-customer-responsibilities.md)
  - [Customize Azure Spring Cloud egress with a User-Defined Route](concept-outbound-type.md)

## Create a VNet instance using a user-defined route

The following illustration shows an example of an Azure Spring Apps VNet instance using a user-defined route.

:::image type="content" source="media/how-to-create-user-defined-route-instance/user-defined-route-example-architecture.png" lightbox="media/how-to-create-user-defined-route-instance/user-defined-route-example-architecture.png" alt-text="Architecture diagram showing user-defined routing.":::

### Set configuration using environment variables

The following example shows how to define a set of environment variables to be used in resource creation.

```bash
PREFIX="asa-egress"
RG="${PREFIX}-rg"
LOC="eastus"
ASANAME="${PREFIX}"
VNET_NAME="${PREFIX}-vnet"
ASA_APP_SUBNET_NAME="asa-app-subnet"
ASA_SERVICE_RUNTIME_SUBNET_NAME="asa-service-runtime-subnet"
# DO NOT CHANGE FWSUBNET_NAME - This is currently a requirement for Azure Firewall.
FWSUBNET_NAME="AzureFirewallSubnet"
FWNAME="${PREFIX}-fw"
FWPUBLICIP_NAME="${PREFIX}-fwpublicip"
FWIPCONFIG_NAME="${PREFIX}-fwconfig"
APP_ROUTE_TABLE_NAME="${PREFIX}-app-rt"
SERVICE_RUNTIME_ROUTE_TABLE_NAME="${PREFIX}-service-runtime-rt"
FWROUTE_NAME="${PREFIX}-fwrn"
ASA_NAME="${PREFIX}-instance"
```

### Create a virtual network with multiple subnets

This section shows you how to provision a virtual network with three separate subnets: one for the user apps, one for service runtime, and one for the firewall.

First create a resource group, as shown in the following example.

```azurecli
# Create resource group.

az group create --name $RG --location $LOC
```

Then create a virtual network with three subnets to host the ASA instance and the Azure Firewall, as shown in the following example.

```azurecli
# Dedicated virtual network with ASA app subnet.

az network vnet create \
    --resource-group $RG \
    --name $VNET_NAME \
    --location $LOC \
    --address-prefixes 10.42.0.0/16 \
    --subnet-name $ASA_APP_SUBNET_NAME \
    --subnet-prefix 10.42.1.0/24

# Dedicated subnet for ASA service runtime subnet.

az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $ASA_SERVICE_RUNTIME_SUBNET_NAME\
    --address-prefix 10.42.2.0/24

# Dedicated subnet for Azure Firewall. (Firewall name cannot be changed.)

az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $FWSUBNET_NAME \
    --address-prefix 10.42.3.0/24
```

### Create and set up an Azure Firewall with a user-defined route

Use the following command to create and set up an Azure Firewall with a user-defined route and configure Azure Firewall outbound rules. The firewall lets you configure granular egress traffic rules from an Azure Spring Apps instance.

> [!IMPORTANT]
> If your cluster or application creates a large number of outbound connections directed to the same or small subset of destinations, you might require more firewall frontend IPs to avoid reaching the maximum ports per front-end IP. For more information on how to create an Azure firewall with multiple IPs, see [Quickstart: Create an Azure Firewall with multiple public IP addresses - ARM template](../firewall/quick-create-multiple-ip-template.md). Create a standard SKU public IP resource that will be used as the Azure Firewall front-end address.

```azurecli
az network public-ip create \
    --resource-group $RG \
    --name $FWPUBLICIP_NAME -l $LOC \
    --sku "Standard"
```

The following example shows how to install the Azure Firewall preview CLI extension and deploy Azure Firewall.

```azurecli
# Install Azure Firewall preview CLI extension.

az extension add --name azure-firewall

# Deploy Azure Firewall.

az network firewall create \
    --resource-group $RG \
    --firewall-name $FWNAME -l $LOC \
    --enable-dns-proxy true
```

The following example shows how to assign the IP address you created to the firewall front end.

> [!NOTE]
> Setting up the public IP address to the Azure Firewall may take a few minutes. To leverage FQDN on network rules, enable DNS proxy. When enabled, the firewall will listen on port 53 and forward DNS requests to the specified DNS server. The firewall can then translate the FQDN automatically.

```azurecli
# Configure firewall IP config.

az network firewall ip-config create \
    --resource-group $RG \
    --firewall-name $FWNAME \
    --name $FWIPCONFIG_NAME \
    --public-ip-address $FWPUBLICIP_NAME \
    --vnet-name $VNET_NAME
```

When the operation has completed, save the firewall front-end IP address for configuration later, as shown in the following example.

```azurecli
# Capture firewall IP address for later use.

FWPUBLIC_IP=$(az network public-ip show \
    --resource-group $RG \
    --name $FWPUBLICIP_NAME \
    --query "ipAddress" \
    --output tsv)
FWPRIVATE_IP=$(az network firewall show \
    --resource-group $RG \
    --name $FWNAME \
    --query "ipConfigurations[0].privateIpAddress" \
    --output tsv | tr -d '[:space:]')
```

### Create a user-defined route with a hop to Azure Firewall

Azure automatically routes traffic between Azure subnets, virtual networks, and on-premises networks. If you want to change Azure's default routing, create a route table.

The following example shows how to create a route table to be associated with a specified subnet. The route table defines the next hop, as in the Azure Firewall you created. Each subnet can have one route table associated with it, or could have no associated route table.

```azurecli
# Create UDR and add a route for Azure Firewall.

az network route-table create \
    --resource-group $RG -l $LOC \
    --name $APP_ROUTE_TABLE_NAME
az network route-table route create \
    --resource-group $RG \
    --name $FWROUTE_NAME \
    --route-table-name $APP_ROUTE_TABLE_NAME \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address $FWPRIVATE_IP
az network route-table create \
    --resource-group $RG -l $LOC \
    --name $SERVICE_RUNTIME_ROUTE_TABLE_NAME
az network route-table route create \
    --resource-group $RG \
    --name $FWROUTE_NAME \
    --route-table-name $SERVICE_RUNTIME_ROUTE_TABLE_NAME \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type VirtualAppliance \
    --next-hop-ip-address $FWPRIVATE_IP
```

### Adding firewall rules

The following example shows hot to add rules to your firewall. For more information, see [Customer responsibilities for running Azure Spring Apps in VNET](vnet-customer-responsibilities.md).

```azurecli
# Add firewall network rules.

az network firewall network-rule create \
    --resource-group $RG \
    --firewall-name $FWNAME \
    --collection-name 'asafwnr' -n 'apiudp' \
    --protocols 'UDP' \
    --source-addresses '*' \
    --destination-addresses "AzureCloud" \
    --destination-ports 1194 \
    --action allow \
    --priority 100
az network firewall network-rule create \
    --resource-group $RG \
    --firewall-name $FWNAME \
    --collection-name 'asafwnr' -n 'springcloudtcp' \
    --protocols 'TCP' \
    --source-addresses '*' \
    --destination-addresses "AzureCloud" \
    --destination-ports 443 445
az network firewall network-rule create \
    --resource-group $RG \
    --firewall-name $FWNAME \
    --collection-name 'asafwnr' \
    --name 'time' \
    --protocols 'UDP' \
    --source-addresses '*' \
    --destination-fqdns 'ntp.ubuntu.com' \
    --destination-ports 123

# Add firewall application rules.

az network firewall application-rule create \
    --resource-group $RG \
    --firewall-name $FWNAME \
    --collection-name 'aksfwar'\
    --name 'fqdn' \
    --source-addresses '*' \
    --protocols 'http=80' 'https=443' \
    --fqdn-tags "AzureKubernetesService" \
    --action allow --priority 100
```

### Associate route tables with subnets

To associate the cluster with the firewall, the dedicated subnet for the cluster's subnet must reference the route table you created. App and service runtime subnets must be associated with corresponding route tables. The following example shows how to associate a route table with a subnet.

```azurecli
# Associate route table with next hop to Firewall to the Azure Spring Apps subnet.

az network vnet subnet update \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $ASA_APP_SUBNET_NAME \
    --route-table $APP_ROUTE_TABLE_NAME

az network vnet subnet update 
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $ASA_SERVICE_RUNTIME_SUBNET_NAME \
    --route-table $SERVICE_RUNTIME_ROUTE_TABLE_NAME
```

### Add a role for an Azure Spring Apps RP

The following example shows how to add a role for an Azure Spring Apps RP.

```azurecli
VIRTUAL_NETWORK_RESOURCE_ID=$(az network vnet show \
    --name $VNET_NAME \
    --resource-group $RG \
    --query "id" \
    --output tsv)

az role assignment create \
    --role "Owner" \
    --scope ${VIRTUAL_NETWORK_RESOURCE_ID} \
    --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2
```

### Create a UDR Azure Spring Apps instance

The following example shows how to create a UDR Azure Spring Apps instance.

```azurecli
az spring create \
    --name $ASA_NAME \
    --resource-group $RG \
    --vnet $VNET_NAME \
    --app-subnet $ASA_APP_SUBNET_NAME \
    --service-runtime-subnet $ASA_SERVICE_RUNTIME_SUBNET_NAME \
    --outbound-type userDefinedRouting
```

You can now access the public IP of the firewall from the internet. The firewall will route traffic into Azure Spring Apps subnets according to your routing rules.

## Next steps

- [Troubleshooting Azure Spring Apps in virtual networks](troubleshooting-vnet.md)
- [Customer responsibilities for running Azure Spring Apps in VNET](vnet-customer-responsibilities.md)
