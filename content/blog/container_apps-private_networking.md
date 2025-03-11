---

title: Azure Container Apps and Private Networking
type: blog
date: 2024-11-11
prev: blog/initial_commit.md
next: blog/dont-bake-app-settings.md
---

# Introduction

Recently, we've been working with setups that require private networking while leveraging Azure Container Apps (ACA). Since ACA is public by default and operates on Kubernetes, figuring out how to implement private networking isn't straightforward—especially given the sparse or perhaps poorly organized documentation. We hope this guide helps you navigate the complexities and get your solution working according to your requirements.

This example leverages Azure Bicep, a tool I strongly advocate for and enjoy using on Azure. I’ve also implemented this code in Terraform, so if anyone has questions, I’m more than happy to help.

## Prerequisites

Most users of Container Apps are likely interested in the Consumption model of the service [(which is incredibly cost-effective)](https://azure.microsoft.com/en-us/pricing/details/container-apps/). We'll demonstrate the necessary configuration for this model initially and also provide a brief overview of the Workload Profile configuration.

Please note that Private Link is not yet supported for the Workload Profile configuration of ACA. This feature is currently in limited private preview and [will be available in the future](https://github.com/microsoft/azure-container-apps/issues/867). However, if your ingress (such as an App Gateway) is in the same virtual network as the ACA, you can still leverage private networking.

## Overview

Below is the layout of the files and modules we'll be working with in this setup:

```plaintext
main.bicep
modules/
  network.bicep
  containerAppEnvironment.bicep
  containerApp.bicep
  logAnalytics.bicep
  privateDnsZone.bicep
```

The `main.bicep` file is the entry point for the deployment. We recommend using a `resourceNames` object to centralize the creation and naming of resources. This approach simplifies management and can help avoid issues with circular dependencies.

**Note regarding the use of the `baseTime` suffix in module names:** This is to assist when debugging module deployments in the Azure Portal. It's not necessary for the deployment to work, but if the module names are identical between deployments, the module names may be overwritten. Including a timestamp helps differentiate between deployments.

```bicep
param location string = resourceGroup().location
@minLength(4)
@maxLength(12)
param rootName string = 'DvPrvNtwk'
param baseTime string = utcNow('u')
param tags object = {
  environment: 'Dv'
  customer: 'Anton_Tech'
  project: 'ACA'
}

var resourceNames = {
  network: rootName
  logAnalytics: '${rootName}Log'
  containerAppsEnvironment: '${rootName}Cae'
  containerApp: '${rootName}Ca'
  privateDnsZone: '${rootName}DnsZone'
  virtualNetworkLink: '${rootName}Vnl'
}

module network './modules/network.bicep' = {
  name: 'network-${baseTime}'
  params: {
    location: location
    name: resourceNames.network
    tags: tags
  }
}

module logAnalytics './modules/logAnalytics.bicep' = {
  name: 'logAnalytics-${baseTime}'
  params: {
    location: location
    name: resourceNames.logAnalytics
    tags: tags
  }
}

module containerAppsEnv './modules/containerAppsEnv.bicep' = {
  name: 'containerAppEnvironment-${baseTime}'
  params: {
    location: location
    name: resourceNames.containerAppsEnvironment
    tags: tags
    logAnalyticsWorkspaceName: logAnalytics.outputs.logAnalyticsWorkspaceName
    containerAppSubnetId: network.outputs.containerAppSubnetId
  }
}

module containerApp './modules/containerApp.bicep' = {
  name: 'containerApp-${baseTime}'
  params: {
    location: location
    name: resourceNames.containerApp
    tags: tags
    containerAppsEnvironmentId: containerAppsEnv.outputs.containerAppsEnvironmentId
    containerImage: 'nginx:latest'
  }
}

module privateDnsZone 'modules/privateDnsZone.bicep' = {
  name: 'privateDnsZone-${baseTime}'
  params: {
    containerAppEnvironmentDefaultDomain: containerAppsEnv.outputs.containerAppsEnvironmentDefaultDomain
    containerAppEnvironmentStaticIp: containerAppsEnv.outputs.containerAppsEnvironmentStaticIp
    virtualNetworkId: network.outputs.virtualNetworkId
    virtualNetworkLinkName: resourceNames.virtualNetworkLink
  }
}
```

## Network Configuration

A few important points in the networking file:

1. The commented-out `delegations` block is provided in case you wish to run Workload Profiles on your [Container App Environment](https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview). The delegation is not necessary when running in the Consumption model but is handy for future reference.

2. With the Consumption model, you must have a minimum subnet address space of `/23`. [See Azure documentation for details.](https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#subnet)

3. The Private Link Service network policy must be disabled on the subnet. Since we will be running the Container App Environment in internal VNet configuration mode, this is necessary.

```bicep
@description('Base name/prefix of all resources')
param name string
param location string
param tags object = {}

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2022-01-01' = {
  name: '${name}Vn'
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
  }
  resource containerAppSubnet 'subnets' = {
    name: '${name}Sn'
    properties: {
      addressPrefix: '10.0.0.0/23'
      delegations: [
        // Example of a delegation for future use:
        // {
        //   name: 'Microsoft.App.environments'
        //   properties: {
        //     serviceName: 'Microsoft.App/environments'
        //   }
        // }
      ]
      networkSecurityGroup: {
        id: subnetNsg.id
      }
      privateLinkServiceNetworkPolicies: 'Disabled'
    }
  }
}

resource subnetNsg 'Microsoft.Network/networkSecurityGroups@2022-01-01' = {
  name: '${name}Nsg'
  location: location
  tags: tags
  properties: {
    securityRules: [
      // Define any necessary security rules here
    ]
  }
}

output containerAppSubnetId string = virtualNetwork.properties.subnets[0].id
output virtualNetworkId string = virtualNetwork.id
```

## Container App Environment

In the Container App Environment (CAE), a couple of important notes:

1. The `staticIp` is necessary for the Private DNS Zone to resolve the FQDN to the ACA. This IP exposes the static IP of the load balancer that fronts the Kubernetes cluster abstracted by the CAE.

2. The `defaultDomain` is exported for use in the Private DNS Zone module. This is used to create a Private DNS Zone with the same name as the default domain of the CAE, which is necessary for any service to resolve the FQDN to the ACA.

```bicep
param name string
param location string 
param containerAppSubnetId string
param logAnalyticsWorkspaceName string
param tags object = {}

resource containerAppEnvironment 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: name
  location: location
  tags: tags
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalyticsWorkspace.properties.customerId
        sharedKey: logAnalyticsWorkspace.listKeys().primarySharedKey
      }
    }
    vnetConfiguration: {
      infrastructureSubnetId: containerAppSubnetId
      internal: true
    }
    workloadProfiles: [
      // Example of a Workload Profile for future use:
      // {
      //   name: 'D4'
      //   workloadProfileType: 'D4'
      //   maximumCount: 10
      //   minimumCount: 1
      // }
    ]
  }
}

resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2021-06-01' existing = {
  name: logAnalyticsWorkspaceName
}

output containerAppsEnvironmentId string = containerAppEnvironment.id
output containerAppsEnvironmentStaticIp string = containerAppEnvironment.properties.staticIp
output containerAppsEnvironmentDefaultDomain string = containerAppEnvironment.properties.defaultDomain
```

## Container App

This is a relatively simple definition, leveraging the `nginx:latest` image for demonstration purposes.

```bicep
param name string
param location string 
param containerAppsEnvironmentId string
param containerImage string
param tags object = {}

resource containerApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: toLower(name)
  location: location
  tags: tags
  properties: {
    managedEnvironmentId: containerAppsEnvironmentId
    configuration: {
      ingress: {
        external: true
        targetPort: 80
      }
    }
    template: {
      containers: [
        {
          name: 'app'
          image: containerImage
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
      }
    }
  }
}

output containerFqdn string = containerApp.properties.configuration.ingress.fqdn
```

## Log Analytics

This is a simple setup for a Log Analytics Workspace.

```bicep
param name string
param location string
param tags object = {}

resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2021-06-01' = {
  name: name
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'PerGB2018'
    }
  }
}

output logAnalyticsWorkspaceId string = logAnalyticsWorkspace.id
output logAnalyticsWorkspaceName string = logAnalyticsWorkspace.name
```

## Private DNS Zone

The Private DNS Zone is the glue that puts everything together. The A record is necessary for the ACA to resolve the FQDN to the static IP of the load balancer for the Container App Environment. Keep the name of the A record as a wildcard `*` to ensure that all subdomains resolve to the same IP. The load balancer will forward the traffic to the appropriate service.

```bicep
param containerAppEnvironmentDefaultDomain string
param containerAppEnvironmentStaticIp string
param virtualNetworkId string
param virtualNetworkLinkName string

// Important: Name the Private DNS Zone the same as the Container App Environment's default domain
// Reference: https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#dns
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: containerAppEnvironmentDefaultDomain
  location: 'global'

  resource virtualNetworkLink 'virtualNetworkLinks' = {
    name: virtualNetworkLinkName
    location: 'global'
    properties: {
      registrationEnabled: false
      virtualNetwork: {
        id: virtualNetworkId
      }
    }
  }

  resource aRecord 'A' = {
    name: '*'
    properties: {
      ttl: 3600
      aRecords: [
        {
          ipv4Address: containerAppEnvironmentStaticIp
        }
      ]
    }
  }
}
```

## Public Ingress Options

At this point, whether you're interested in exposing the public ingress with Azure Application Gateway or Azure Front Door (or another service), you can now set up and access your privately networked Container App Environment. Whichever service you choose, ensure the following:

1. The service is in the same virtual network as the ACA.
2. If the service is outside of the virtual network, create a new Virtual Network Link in the Private DNS Zone module for the service's virtual network. This allows the service to resolve the FQDN to the ACA.
3. The service is configured to route traffic to the ACA's FQDN.

# Conclusion

Overall, the setup is straightforward once you know what to look for. The documentation may be sparse at the moment, but we hope this guide helps you get started with your ACA private networking setup. If there's interest, we'd be happy to provide a follow-up on how to set up public ingress with Azure Application Gateway or Azure Front Door.

If you have any questions or need further clarification, feel free to reach out to us at Anton Tech. We're always happy to help! 🚀
