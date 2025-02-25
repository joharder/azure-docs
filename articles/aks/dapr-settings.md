---
title: Configure the Dapr extension for your Azure Kubernetes Service (AKS) and Arc-enabled Kubernetes project
description: Learn how to configure the Dapr extension specifically for your Azure Kubernetes Service (AKS) and Arc-enabled Kubernetes project
author: hhunter-ms
ms.author: hannahhunter
ms.service: container-service
ms.topic: article
ms.date: 01/09/2023
---

# Configure the Dapr extension for your Azure Kubernetes Service (AKS) and Arc-enabled Kubernetes project

Once you've [created the Dapr extension](./dapr.md), you can configure the [Dapr](https://dapr.io/) extension to work best for you and your project using various configuration options, like:

- Limiting which of your nodes use the Dapr extension
- Setting automatic CRD updates
- Configuring the Dapr release namespace

The extension enables you to set Dapr configuration options by using the `--configuration-settings` parameter. For example, to provision Dapr with high availability (HA) enabled, set the `global.ha.enabled` parameter to `true`:

```azurecli
az k8s-extension create --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=2"
```

> [!NOTE]
> If configuration settings are sensitive and need to be protected, for example cert related information, pass the `--configuration-protected-settings` parameter and the value will be protected from being read.

If no configuration-settings are passed, the Dapr configuration defaults to:

```yaml
  ha:
    enabled: true
    replicaCount: 3
    disruption:
      minimumAvailable: ""
      maximumUnavailable: "25%"
  prometheus:
    enabled: true
    port: 9090
  mtls:
    enabled: true
    workloadCertTTL: 24h
    allowedClockSkew: 15m
```

For a list of available options, see [Dapr configuration][dapr-configuration-options].

## Limiting the extension to certain nodes

In some configurations, you may only want to run Dapr on certain nodes. You can limit the extension by passing a `nodeSelector` in the extension configuration. If the desired `nodeSelector` contains `.`, you must escape them from the shell and the extension. For example, the following configuration will install Dapr to only nodes with `topology.kubernetes.io/zone: "us-east-1c"`:

```azurecli
az k8s-extension create --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=2" \
--configuration-settings "global.nodeSelector.kubernetes\.io/zone: us-east-1c"
```

For managing OS and architecture, use the [supported versions](https://github.com/dapr/dapr/blob/b8ae13bf3f0a84c25051fcdacbfd8ac8e32695df/docker/docker.mk#L50) of the `global.daprControlPlaneOs` and `global.daprControlPlaneArch` configuration:

```azurecli
az k8s-extension create --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=2" \
--configuration-settings "global.daprControlPlaneOs=linux” \
--configuration-settings "global.daprControlPlaneArch=amd64”
```
## Configure the Dapr release namespace

You can configure the release namespace. The Dapr extension gets installed in the `dapr-system` namespace by default. To override it, use `--release-namespace`. Include the cluster `--scope` to redefine the namespace.

```azurecli
az k8s-extension create \
--cluster-type managedClusters \
--cluster-name dapr-aks \
--resource-group dapr-rg \
--name my-dapr-ext \
--extension-type microsoft.dapr \
--release-train stable \
--auto-upgrade false \
--version 1.9.2 \
--scope cluster \
--release-namespace dapr-custom
```

[Learn how to configure the Dapr release namespace if you already have Dapr installed](./dapr-migration.md).

## Show current configuration settings

Use the `az k8s-extension show` command to show the current Dapr configuration settings:  

```azurecli
az k8s-extension show --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr
```

## Update configuration settings

> [!IMPORTANT]
> Some configuration options cannot be modified post-creation. Adjustments to these options require deletion and recreation of the extension, applicable to the following settings:
> * `global.ha.*`
> * `dapr_placement.*`
>
> HA is enabled enabled by default. Disabling it requires deletion and recreation of the extension.

To update your Dapr configuration settings, recreate the extension with the desired state. For example, assume we've previously created and installed the extension using the following configuration:

```azurecli-interactive
az k8s-extension create --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \  
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=2" 
```

To update the `dapr_operator.replicaCount` from two to three, use the following command:

```azurecli-interactive
az k8s-extension create --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=3"
```

## Set the outbound proxy for Dapr extension for Azure Arc on-premises

If you want to use an outbound proxy with the Dapr extension for AKS, you can do so by:

1. Setting the proxy environment variables using the [`dapr.io/env` annotations](https://docs.dapr.io/reference/arguments-annotations-overview/):
   - `HTTP_PROXY`
   - `HTTPS_PROXY`
   - `NO_PROXY`
1. [Installing the proxy certificate in the sidecar](https://docs.dapr.io/operations/configuration/install-certificates/).

## Disable automatic CRD updates

With Dapr version 1.9.2, CRDs are automatically upgraded when the extension upgrades. To disable this setting, you can set `hooks.applyCrds` to `false`. 

```azurecli
az k8s-extension upgrade --cluster-type managedClusters \
--cluster-name myAKSCluster \
--resource-group myResourceGroup \
--name dapr \
--extension-type Microsoft.Dapr \
--auto-upgrade-minor-version true \
--configuration-settings "global.ha.enabled=true" \
--configuration-settings "dapr_operator.replicaCount=2" \
--configuration-settings "global.daprControlPlaneOs=linux” \
--configuration-settings "global.daprControlPlaneArch=amd64” \
--configuration-settings "hooks.applyCrds=false"
```

> [!NOTE]
> CRDs are only applied in case of upgrades and are skipped during downgrades.


## Meet network requirements

The Dapr extension for AKS and Arc for Kubernetes requires outbound URLs on `https://:443` to function. In addition to the `https://mcr.microsoft.com/daprio` URL for pulling Dapr artifacts, verify you've included the [outbound URLs required for AKS or Arc for Kubernetes](../azure-arc/kubernetes/quickstart-connect-cluster.md#meet-network-requirements). 

## Next Steps

Once you have successfully provisioned Dapr in your AKS cluster, try deploying a [sample application][sample-application].

<!-- LINKS INTERNAL -->
[deploy-cluster]: ./tutorial-kubernetes-deploy-cluster.md
[az-feature-register]: /cli/azure/feature#az-feature-register
[az-feature-list]: /cli/azure/feature#az-feature-list
[az-provider-register]: /cli/azure/provider#az-provider-register
[sample-application]: ./quickstart-dapr.md
[k8s-version-support-policy]: ./supported-kubernetes-versions.md?tabs=azure-cli#kubernetes-version-support-policy
[arc-k8s-cluster]: ../azure-arc/kubernetes/quickstart-connect-cluster.md
[update-extension]: ./cluster-extensions.md#update-extension-instance
[install-cli]: /cli/azure/install-azure-cli
[dapr-migration]: ./dapr-migration.md
[dapr-settings]: ./dapr-settings.md

<!-- LINKS EXTERNAL -->
[kubernetes-production]: https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-production
[building-blocks-concepts]: https://docs.dapr.io/developing-applications/building-blocks/
[dapr-configuration-options]: https://github.com/dapr/dapr/blob/master/charts/dapr/README.md#configuration
[sample-application]: https://github.com/dapr/quickstarts/tree/master/hello-kubernetes#step-2---create-and-configure-a-state-store
[dapr-security]: https://docs.dapr.io/concepts/security-concept/
[dapr-deployment-annotations]: https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-overview/#adding-dapr-to-a-kubernetes-deployment
[dapr-oss-support]: https://docs.dapr.io/operations/support/support-release-policy/
[dapr-supported-version]: https://docs.dapr.io/operations/support/support-release-policy/#supported-versions
[dapr-troubleshooting]: https://docs.dapr.io/operations/troubleshooting/common_issues/
[supported-cloud-regions]: https://azure.microsoft.com/global-infrastructure/services/?products=azure-arc
