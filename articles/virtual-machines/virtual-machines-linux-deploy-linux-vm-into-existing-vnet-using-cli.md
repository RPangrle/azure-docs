---
title: Deploy a Linux VM into an existing Azure Virtual Network using the CLI | Microsoft Docs
description: Deploy a Linux VM into an existing Virtual Network using the CLI.
services: virtual-machines-linux
documentationcenter: virtual-machines
author: vlivech
manager: timlt
editor: ''
tags: azure-resource-manager

ms.assetid:
ms.service: virtual-machines-linux
ms.workload: infrastructure
ms.tgt_pltfrm: vm-linux
ms.devlang: na
ms.topic: article
ms.date: 11/30/2016
ms.author: v-livech

---

# Deploy a Linux VM into an existing VNet & NSG using the CLI

This article shows how to use CLI flags to deploy a VM into an existing Virtual Network (VNet), that is secured with an existing Network Security Group (NSG).  The requirements are:

- [an Azure account](https://azure.microsoft.com/pricing/free-trial/)

- [SSH public and private key files](virtual-machines-linux-mac-create-ssh-keys.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

## Quick Commands

Replace any examples with your own settings.

### Create the Resource Group

```azurecli
azure group create myResourceGroup \
-l westus
```

### Create the VNet

```azurecli
azure network vnet create myVNet \
-g myResourceGroup \
-a 10.10.0.0/24 \
-l westus
```

### Create the NSG

```azurecli
azure network nsg create myNSG \
-g myResourceGroup \
-l westus
```

### Add an inbound SSH allow rule

```azurecli
azure network nsg rule create inboundSSH \
-g myResourceGroup \
-a myNSG \
-c Allow \
-p Tcp \
-r Inbound \
-y 100 \
-f Internet \
-o 22 \
-e 10.10.0.0/24 \
-u 22
```

### Add a subnet to the VNet

```azurecli
azure network vnet subnet create mySubNet \
-g myResourceGroup \
-e myVNet \
-a 10.10.0.0/26 \
-o myNSG
```

### Add a VNic to the subnet

```azurecli
azure network nic create myVNic \
-g myResourceGroup \
-l westus \
-m myVNet \
-k mySubNet
```

### Deploy the VM into the VNet, NSG and connect the VNic

```azurecli
azure vm create myVM \
-g myResourceGroup \
-l westus \
-y linux \
-Q Debian \
-o myStorageAcct \
-u myAdminUser \
-M ~/.ssh/id_rsa.pub \
-n myVM \
-F myVNet \
-j mySubnet \
-N myVNic
```

## Detailed walkthrough

It is recommended that Azure assets like the VNets and NSGs should be static and long lived resources that are rarely deployed.  Once a VNet has been deployed, it can be reused by new deployments without any adverse affects to the infrastructure.  Think about a VNet as being a traditional hardware network switch, you would not need to configure a brand new hardware switch with each deployment.  With a correctly configured VNet, we can continue to deploy new servers into that VNet over and over with few, if any, changes required over the life of the VNet.

## Create the Resource group

First we will create a Resource Group to organize everything we create in this walkthrough.  For more information on Azure Resource Groups, see [Azure Resource Manager overview](../azure-resource-manager/resource-group-overview.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

```azurecli
azure group create myResourceGroup \
--location westus
```

## Create the VNet

The first step is to build a VNet to launch the VMs into.  The VNet contains one subnet for this walkthrough.  For more information on Azure VNets, see [Create a virtual network by using the Azure CLI](../virtual-network/virtual-networks-create-vnet-arm-cli.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

```azurecli
azure network vnet create myVNet \
--resource-group myResourceGroup \
--address-prefixes 10.10.0.0/24 \
--location westus
```

## Create the NSG

The Subnet is built behind an existing Network Security Group so we build the NSG before the Subnet.  Azure NSGs are equivalent to a firewall at the network layer.  For more information on Azure NSGs, see [How to create NSGs in the Azure CLI](../virtual-network/virtual-networks-create-nsg-arm-cli.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

```azurecli
azure network nsg create myNSG \
--resource-group myResourceGroup \
--location westus
```

## Add an inbound SSH allow rule

The Linux VM needs access from the internet so a rule allowing inbound port 22 traffic to be passed through the network to port 22 on the Linux VM is needed.

```azurecli
azure network nsg rule create inboundSSH \
--resource-group myResourceGroup \
--nsg-name myNSG \
--access Allow \
--protocol Tcp \
--direction Inbound \
--priority 100 \
--source-address-prefix Internet \
--source-port-range 22 \
--destination-address-prefix 10.10.0.0/24 \
--destination-port-range 22
```

## Add a subnet to the VNet

VMs within the VNet must be located in a subnet.  Each VNet can have multiple subnets.  Create the subnet and associate the subnet with the NSG to add a firewall to the subnet.

```azurecli
azure network vnet subnet create mySubNet \
--resource-group myResourceGroup \
--vnet-name myVNet \
--address-prefix 10.10.0.0/26 \
--network-security-group-name myNSG
```

The Subnet is now added inside the VNet and associated with the NSG and the NSG rule.


## Add a VNic to the subnet

Virtual network cards (VNics) are important as you can reuse them by connecting them to different VMs, which keeps the VNic as a static resource while the VMs can be temporary.  Create a VNic and associate it with the Subnet created in the previous step.

```azurecli
azure network nic create myVNic \
-g myResourceGroup \
-l westus \
-m myVNet \
-k mySubNet
```

## Deploy the VM into the VNet and NSG

We now have a VNet, a subnet inside that VNet, and an NSG acting as a firewall to protect our subnet by blocking all inbound traffic except port 22 for SSH.  The VM can now be deployed inside this existing network infrastructure.

Using the Azure CLI, and the `azure vm create` command, the Linux VM is deployed to the existing Azure Resource Group, VNet, Subnet, and VNic.  For more information on using the CLI to deploy a complete VM, see [Create a complete Linux environment by using the Azure CLI](virtual-machines-linux-create-cli-complete.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

```azurecli
azure vm create myVM \
--resource-group myResourceGroup \
--location westus \
--os-type linux \
--image-urn Debian \
--storage-account-name mystorageaccount \
--admin-username myAdminUser \
--ssh-publickey-file ~/.ssh/id_rsa.pub \
--vnet-name myVNet \
--vnet-subnet-name mySubnet \
--nic-name myVNic
```

By using the CLI flags to call out existing resources, we instruct Azure to deploy the VM inside the existing network.  To reiterate, once a VNet and subnet have been deployed, they can be left as static or permanent resources inside your Azure region.  

## Next steps

* [Use an Azure Resource Manager template to create a specific deployment](virtual-machines-linux-cli-deploy-templates.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Create your own custom environment for a Linux VM using Azure CLI commands directly](virtual-machines-linux-create-cli-complete.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Create a Linux VM on Azure using templates](virtual-machines-linux-create-ssh-secured-vm-from-template.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
