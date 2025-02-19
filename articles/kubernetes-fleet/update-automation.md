---
title: "Automate upgrades of Kubernetes and node images across multiple clusters using Azure Kubernetes Fleet Manager"
description: Learn how to configure automated upgrades of Kubernetes and node images across multiple clusters by using Azure Kubernetes Fleet Manager.
ms.topic: how-to
ms.date: 09/16/2024
author: sjwaight
ms.author: simonwaight
ms.service: azure-kubernetes-fleet-manager
---

# Automate upgrades of Kubernetes and node images across multiple clusters using Azure Kubernetes Fleet Manager (preview)

Platform admins managing large number of clusters often have problems with staging the updates of multiple clusters (for example, upgrading node OS image or Kubernetes versions) in a safe and predictable way. To address this challenge, Azure Kubernetes Fleet Manager (Fleet) allows you to orchestrate updates across multiple clusters using update runs.

Update runs consist of stages, groups, and strategies and can be applied either manually, for one-time updates, or automatically, for ongoing regular updates using auto-upgrade profiles. All update runs (manual or automated) honor member cluster maintenance windows.

This article covers how to use auto-upgrade profiles to automatically trigger update runs when new Kubernetes or node image versions are made available. 

[!INCLUDE [preview features note](./includes/preview/preview-callout.md)]

## Prerequisites

* Read the [conceptual overview of auto-upgrade profiles](./concepts-update-orchestration.md#understanding-auto-upgrade-profiles-preview), which provides an explanation of configurations referenced in this guide.

* You must have a Fleet resource with one or more member clusters. If not, follow the [quickstart][fleet-quickstart] to create a Fleet resource and join Azure Kubernetes Service (AKS) clusters as members.

* If you wish to use an update strategy you should configure one using the instructions in the [update run how-to article](./update-orchestration.md#create-an-update-run-using-update-strategies). You need the update strategy resource identifier to use with an auto-upgrade profile.

* Set the following environment variables:

    ```bash
    export GROUP=<resource-group>
    export FLEET=<fleet-name>
    export AUTOUPGRADEPROFILE=<upgrade-profile-name>
    # Optional
    export STRATEGYID=<strategy-id>
    export CLUSTER=<ask-cluster-name>
    ```

* You need Azure CLI version 2.61.0 or later installed. To install or upgrade, see [Install the Azure CLI][azure-cli-install].

* You also need the `fleet` Azure CLI extension version 1.3.0 or later, which you can install by running the following command:

  ```azurecli-interactive
  az extension add --name fleet
  ```

  Run the following command to update to the latest version of the extension released:

  ```azurecli-interactive
  az extension update --name fleet
  ```

> [!NOTE]
> Auto-upgrade triggered update runs honor [planned maintenance windows](/azure/aks/planned-maintenance) that you set at the AKS cluster level. For more information, see [planned maintenance across multiple member clusters](./concepts-update-orchestration.md#planned-maintenance) which explains how update runs handle member clusters that have been configured with planned maintenance windows.

## Create auto-upgrade profiles  

Use the [`az fleet autoupgradeprofile create`][az-fleet-autoupgradeprofile-create] command to create profiles as shown.

You can create a disabled auto-upgrade profile by passing the `--disabled` argument when using the `create` command. In order to enable the auto-upgrade profile, you must reissue the entire `create` command and omit the `--disabled` argument.

### Stable channel Kubernetes updates

Update to the latest supported Kubernetes patch release on minor version N-1, where N is the latest supported minor version.

Update member clusters sequentially one-by-one.

```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --channel Stable
```

Update member clusters using an existing update strategy.
 
```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --update-strategy-id $STRATEGYID \
  --channel Stable
```

Update member clusters using an existing update strategy, ensuring the same node image version is used in every Azure region. Member clusters will all run the same node image version.
 
```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --update-strategy-id $STRATEGYID \
  --channel Stable \
  --node-image-selection-type Consistent
```

Update member clusters using an existing update strategy, using the latest available node image version for each Azure region. Member clusters may run multiple node image versions. 
 
```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --update-strategy-id $STRATEGYID \
  --channel Stable \
  --node-image-selection-type Latest
```

### Node image updates

Update nodes with a newly patched VHD containing security fixes and bug fixes.

Update node images for member clusters, processing clusters sequentially one-by-one.

```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --channel NodeImage
```

Update node images for member clusters, processing clusters using an existing update strategy.

```azurecli-interactive
az fleet autoupgradeprofile create \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE \
  --update-strategy-id $STRATEGYID \
  --channel NodeImage 
```

## View auto-upgrade profile

You can use the `autoupgradeprofile list` or `autoupgradeprofile show` commands to view the auto-upgrade profile.

List all auto-upgrade profiles for a Fleet.

```azurecli-interactive
az fleet autoupgradeprofile list \
  --resource-group $GROUP \
  --fleet-name $FLEET
```

Show a specific auto-upgrade profile for a Fleet.

```azurecli-interactive
az fleet autoupgradeprofile list \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE
```

## Delete auto-upgrade profile

Use the following command to delete an existing auto-upgrade profile. You are asked to confirm the deletion. If you wish to immediately delete the profile, include `--yes`.

```azurecli-interactive
az fleet autoupgradeprofile delete \
  --resource-group $GROUP \
  --fleet-name $FLEET \
  --name $AUTOUPGRADEPROFILE
```

## Validate auto-upgrade

Auto-upgrades will happen only when new Kubernetes or node images are made available. You can check your existing versions as a baseline as follows.

```azurecli-interactive
# Get Kubernetes version for a member cluster
az aks show \
  --resource-group $GROUP \
  --name $CLUSTER \
  --query currentKubernetesVersion
```

```azurecli-interactive
# Get NodeImage version for a member cluster
az aks show \
  --resource-group $GROUP \
  --name $CLUSTER \
  --query "agentPoolProfiles[].{name:name,mode:mode, nodeImageVersion:nodeImageVersion, osSku:osSku, osType:osType}"
```

Once update runs have completed, you can rerun these commands and view the updated versions that are deployed.

---

<!-- INTERNAL LINKS -->
[az-fleet-member-create]: /cli/azure/fleet/member#az-fleet-member-create
[az-fleet-autoupgradeprofile-create]: /cli/azure/fleet/autoupgradeprofile#az-fleet-autoupgradeprofile-create
[azure-cli-install]: /cli/azure/install-azure-cli

<!-- LINKS -->
[fleet-quickstart]: quickstart-create-fleet-and-members.md
[quickstart-create-fleet]: quickstart-create-fleet-and-members.md#create-a-fleet-resource
[quickstart-create-fleet-member]: quickstart-create-fleet-and-members.md#join-member-clusters
[concepts-update-orchestration]:concepts-update-orchestration.md#update-orchestration-across-multiple-member-clusters