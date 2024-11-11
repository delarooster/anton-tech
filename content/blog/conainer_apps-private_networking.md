---
title: Azure Container Apps and Private Networking
type: blog
prev: blog/initial_commit.md
# next: blog/bicep.md
---

# Introduction
We've been working recently with setups that require private networking while leveraging Azure Container Apps (ACA).  Given that by default, ACA is public and rides on the backbone of Kubernetes, how one goes about leveraging such a solution isn't exactly obvious given that the documentation is still a bit sparse (perhaps just poorly organized?  I'll let you decide).  

Hopefully this helps you get through the weeds and get your solution working as fits your requirements.

## Table of Contents
- [Introduction](#introduction)
  - [Table of Contents](#table-of-contents)
  - [Stipulations](#stipulations)
  - [Overview](#overview)
  - [Network](#network)
  - [Container App Environment](#container-app-environment)
  - [Container App](#container-app)
  - [Log Analytics](#log-analytics)
  - [Private DNS Zone](#private-dns-zone)
  - [Public Ingress](#public-ingress)
- [Conclusion](#conclusion)

## Stipulations
Most people that leverage Container Apps are most likely looking for the Consumption model of the service ([incredibly cost effective!](https://azure.microsoft.com/en-us/pricing/details/container-apps/)).  We'll demonstrate the necessary configuration for this model initially and also provide a brief overview of the Workload Profile configuration as well.

Also note that in the case of the Workload Profile configuration for ACA Private Link is not yet supported.  This is a feature that is currently in limited private preview and [will be available in the future](https://github.com/microsoft/azure-container-apps/issues/867).  However, should your ingress (such as an App Gateway) be in the same virtual network as the ACA, you can still easily leverage private networking.

## Overview
This is the layout of the files and modules that we'll be working with in this setup.

```plaintext
main.bicep
modules/
  network.bicep
  containerAppEnvironment.bicep
  containerApp.bicep
  logAnalytics.bicep
  privateDnsZone.bicep
```

This file is the entry point for the deployment.  I recommend leveraging the use of a `resourceNames` object for ease of creating and naming resources in one place.  This also helps when you're backed into corners of circular dependencies.

N.B. regarding the use of `baseTime` suffix in the module names: this is merely to assist when debugging module deployments in the Azure Portal. It's not necessary for the deployment to work but if the module names are identical between deployments then the module names will be overwritten.  Take it or leave it, I find it handy!

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

## Network
A few points in this networking file:

1. The commented out `delegations` block is in place should you wish to run Worload Profiles on your [Container App Environment.](https://learn.microsoft.com/en-us/azure/container-apps/workload-profiles-overview)  The delegation is not necessary when running as Consumption, but handy for future reference.
2. With Consumption model you must have a minimum of `/23` for [the subnet address space.](https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#subnet)
3. Private Link Service Network Policy must be disabled on the subnet.  We will be running the Container App Environment in Internal Vnet Configuration mode, so this is necessary.
```bicep
@description('Basename / Prefix of all resources')
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
        // as an example of a delegation
        // {
        //   name: 'Microsoft.App.environments'
        //   properties: {
        //     serviceName: 'Microsoft.App/environments'
        //   }
        //   type: 'Microsoft.Network/virtualNetworks/subnets/delegations'
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
    ]
  }
}

output containerAppSubnetId string = virtualNetwork.properties.subnets[0].id
output virtualNetworkId string = virtualNetwork.id
```

## Container App Environment
In the CAE a couple callouts:

1. The Static IP is necessary for the Private DNS Zone to resolve the FQDN to the ACA.  This IP exposes the static ip of the load balancer that fronts the Kubernetes cluster that the CAE abstracts.
2. The CAE Default Domain is exported for use in the Private DNS Zone module.  This is used to create a Private DNS Zone with the same name as the Default Domain of the CAE, which is necessary for any service to resolve the FQDN to the ACA.
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
      // as an example of a Workload Profile
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
Relatively simple definition here...
just leveraging the `nginx:latest` image for demonstration purposes.
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
Who doesn't love a little logging?  This is a simple setup for a Log Analytics Workspace.
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
This Private DNS Zone is the glue that puts everything together.  The A Record is necessary for the ACA to resolve the FQDN to the static IP of the load balancer for the Container App Environment.  Keep the name of the A record a wildcard `*` to ensure that all subdomains resolve to the same IP, the load balancer will (obviously) forward the traffic as required to the appropriate service.
```bicep
param containerAppEnvironmentDefaultDomain string
param containerAppEnvironmentStaticIp string
param virtualNetworkId string
param virtualNetworkLinkName string

// ! Subtle, yet important, for proper resolution to name the Private Dns Zone the same as the Container Apps Environment Default Domain
// ! https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#dns
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

## Public Ingress
At this point whether you're interested in exposing the Public ingress with Azure App Gateway or Azure Front Door (or something else!), now you can setup and access privately networked Container App Environment.  Whichever service you choose you'll want to ensure:
1. The service is in the same virtual network as the ACA.
2. If the service is outside of the virtual network, you'll want to create a new Virtual Network Link in the Private DNS Zone module for the service's virtual network.  This is how the service will resolve the FQDN to the ACA.
3. The service is configured to route traffic to the ACA's FQDN.

# Conclusion
All in all the setup is fairly straightforward once you know what you're looking for.  The documentation is a bit sparse at the moment, but hopefully this guide will help you get started with your ACA private networking setup.  Should there be interest I'll be happy to provide a follow-up on how to setup the Public Ingress with Azure App Gateway or Azure Front Door.

If you have any questions or need further clarification, feel free to reach out to us at Anton Tech.  We're always happy to help! 🚀
