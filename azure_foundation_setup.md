# Phase 1: Azure Foundation Setup

This document outlines the foundational Azure setup for our project. It includes details on resource group structure, region selection, networking design, SKU selections for key Azure services, and conceptual Bicep scripts for deployment.

## 1. Resource Group Structure

We will adopt a modular resource group strategy to organize resources based on their lifecycle and purpose.

*   **`rg-core-infra-<env>`**: This resource group will host core infrastructure components like the virtual network (VNet), Network Security Groups (NSGs), private DNS zones, and Key Vault.
*   **`rg-ai-services-<env>`**: This resource group will contain Azure AI Studio instances, associated storage accounts, and Cognitive Services.
*   **`rg-app-hosting-<env>`**: This resource group will house Azure Container Apps environment, Container Apps, and related resources like Log Analytics workspace.
*   **`rg-gpu-compute-<env>`**: This resource group will be dedicated to GPU Virtual Machines used for intensive model training or processing.

Replace `<env>` with the environment identifier (e.g., `dev`, `staging`, `prod`).

## 2. Region Selection

*   **Primary Region**: `East US 2` - Chosen for its wide range of service availability, including latest GPU SKUs and AI services.
*   **Secondary Region (for DR)**: `Central US` - Selected for geographic separation and availability of complementary services.

All resources within an environment will initially be deployed to the primary region. Disaster Recovery strategies will leverage the secondary region.

## 3. Networking Design

### 3.1. Virtual Network (VNet)

*   **Name**: `vnet-main-<env>-<region_shortcode>` (e.g., `vnet-main-dev-eus2`)
*   **Address Space**: `10.X.0.0/16` (e.g., `10.1.0.0/16` for dev, `10.2.0.0/16` for staging)

### 3.2. Subnets

*   **`snet-containerapps-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.1.0/24`
    *   Purpose: For Azure Container Apps Environment.
    *   Service Delegation: `Microsoft.App/environments`
*   **`snet-aistudio-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.2.0/24`
    *   Purpose: For Azure AI Studio related resources (e.g., private endpoints for storage).
*   **`snet-gpuvms-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.3.0/24`
    *   Purpose: For GPU Virtual Machines.
*   **`snet-privateendpoints-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.4.0/24`
    *   Purpose: Dedicated subnet for private endpoints.
*   **`snet-appgateway-<env>-<region_shortcode>` (Optional)**:
    *   Address Prefix: `10.X.5.0/26`
    *   Purpose: If Azure Application Gateway is used for ingress.
*   **`AzureBastionSubnet` (Optional)**:
    *   Address Prefix: `10.X.6.0/27`
    *   Purpose: If Azure Bastion is used for secure VM access.

### 3.3. Network Security Groups (NSGs)

*   NSGs will be applied at the subnet level.
*   Default NSG rules will be restrictive, only allowing necessary traffic.
*   Specific rules:
    *   **`nsg-containerapps`**: Allow necessary traffic for Container Apps (e.g., HTTPS, internal VNet communication).
    *   **`nsg-aistudio`**: Allow traffic related to AI Studio operations and private endpoint communication.
    *   **`nsg-gpuvms`**: Allow SSH/RDP (if needed, ideally via Bastion) and traffic for data transfer and model training.
    *   **`nsg-privateendpoints`**: Typically allows all traffic from within the VNet but denies all from outside.

### 3.4. Private Endpoints

Private Endpoints will be used extensively to secure access to PaaS services:
*   Azure AI Studio and its associated Storage Account, Key Vault, Container Registry.
*   Azure Container Registry (if used for custom images).
*   Azure Key Vault.
*   Azure Storage Accounts used by various services.

### 3.5. DNS

*   **Private DNS Zones**:
    *   `privatelink.blob.core.windows.net`
    *   `privatelink.file.core.windows.net`
    *   `privatelink.queue.core.windows.net`
    *   `privatelink.table.core.windows.net`
    *   `privatelink.vaultcore.azure.net` (Key Vault)
    *   `privatelink.api.azureml.ms` (Azure AI Studio/Azure Machine Learning)
    *   `privatelink.<region>.inference.ml.azure.com` (Azure AI Studio/Azure Machine Learning inference)
    *   `privatelink.notebooks.azure.net` (Azure AI Studio/Azure Machine Learning notebooks)
    *   `privatelink.azurecontainerapps.io` (Azure Container Apps Environment)
*   These zones will be linked to the main VNet (`vnet-main-<env>-<region_shortcode>`).

## 4. SKU Selections

### 4.1. Azure Container Apps Environment

*   **Plan**: Consumption + Dedicated plan group.
    *   Start with Consumption for general workloads.
    *   Utilize Dedicated plan instances for workloads requiring more consistent performance or specific hardware (e.g., memory-optimized).
*   SKU choice within Dedicated plan will depend on specific app requirements (e.g., `Standard_D4_v5`).

### 4.2. Azure AI Studio

*   Azure AI Studio itself doesn't have a "SKU" in the traditional sense. It's a platform that orchestrates other Azure services.
*   **Associated Azure Machine Learning Workspace**: Standard tier.
*   **Associated Storage Account**: `Standard_LRS` or `Standard_GRS` (for DR). Consider `Premium_LRS` if blob access performance is critical for datasets.
*   **Associated Key Vault**: Standard tier.
*   **Associated Application Insights**: Basic or Standard, depending on logging and monitoring needs.
*   **Associated Container Registry**: Basic for initial development, Standard or Premium for production workloads with geo-replication needs.

### 4.3. GPU Virtual Machines

*   **Series**: NCasT4_v3-series, NC_A100_v4 series, or ND_A100_v4 series.
    *   **`NCasT4_v3-series` (e.g., `Standard_NC4as_T4_v3`)**: Good for smaller training jobs, inference, or development.
    *   **`NC_A100_v4 series` or `ND_A100_v4 series`**: For large-scale training and demanding HPC workloads. Choice depends on memory and interconnect requirements.
*   **OS Disk**: Premium SSD for better performance.
*   **Storage**: Premium SSD or Ultra Disk for datasets and model checkpoints, depending on IOPS and throughput requirements.

## 5. Conceptual Bicep Script Outlines

The following Bicep files provide a conceptual structure for deploying the foundation.

### `main.bicep`

```bicep
// main.bicep
targetScope = 'subscription' // or resourceGroup if deploying RGs manually first

param environment string = 'dev'
param location string = 'eastus2'
param vnetAddressPrefix string = '10.1.0.0/16'

var resourceGroupNameCore = 'rg-core-infra-${environment}'
var resourceGroupNameAI = 'rg-ai-services-${environment}'
var resourceGroupNameApps = 'rg-app-hosting-${environment}'
var resourceGroupNameGPU = 'rg-gpu-compute-${environment}'

// Resource Group Definitions
resource rgCore 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameCore
  location: location
}

resource rgAI 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameAI
  location: location
}

resource rgApps 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameApps
  location: location
}

resource rgGPU 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameGPU
  location: location
}

// Core Infrastructure Module
module network 'modules/network.bicep' = {
  scope: resourceGroup(resourceGroupNameCore)
  name: 'networkDeployment'
  params: {
    vnetName: 'vnet-main-${environment}-${substring(location, 0, 4)}'
    vnetAddressPrefix: vnetAddressPrefix
    location: location
    environment: environment
  }
  dependsOn: [rgCore]
}

module keyvault 'modules/keyvault.bicep' = {
  scope: resourceGroup(resourceGroupNameCore)
  name: 'keyvaultDeployment'
  params: {
    kvName: 'kv-main-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    vnetId: network.outputs.vnetId
    privateEndpointSubnetId: network.outputs.privateEndpointSubnetId
  }
  dependsOn: [rgCore, network]
}

// AI Services Module
module aiStudio 'modules/aistudio.bicep' = {
  scope: resourceGroup(resourceGroupNameAI)
  name: 'aiStudioDeployment'
  params: {
    aiStudioName: 'aistudio-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    // Pass necessary IDs from network and keyvault modules
    vnetId: network.outputs.vnetId 
    privateEndpointSubnetId: network.outputs.privateEndpointSubnetId
    keyVaultId: keyvault.outputs.keyVaultId
  }
  dependsOn: [rgAI, network, keyvault]
}

// App Hosting Module
module containerAppsEnv 'modules/containerapps_env.bicep' = {
  scope: resourceGroup(resourceGroupNameApps)
  name: 'containerAppsEnvDeployment'
  params: {
    caeName: 'cae-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    vnetId: network.outputs.vnetId
    controlPlaneSubnetId: network.outputs.containerAppsSubnetId // Assuming specific subnet for CAE control plane if different
    appSubnetId: network.outputs.containerAppsSubnetId
    logAnalyticsWorkspaceId: null // Placeholder, ideally create/reference LA workspace
    keyVaultId: keyvault.outputs.keyVaultId
  }
  dependsOn: [rgApps, network, keyvault]
}

// GPU VMs module would be similar, defining VMs within rg-gpu-compute-<env>
// module gpuVms 'modules/gpuvms.bicep' = { ... }

output coreRgName string = resourceGroupNameCore
output aiRgName string = resourceGroupNameAI
output appsRgName string = resourceGroupNameApps
output gpuRgName string = resourceGroupNameGPU
output vnetId string = network.outputs.vnetId
```

### `modules/network.bicep`

```bicep
// modules/network.bicep
param vnetName string
param vnetAddressPrefix string
param location string
param environment string

// Define subnet configurations as an array of objects
var subnets = [
  {
    name: 'snet-containerapps-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 1) // 10.X.1.0/24
    delegations: [
      {
        name: 'Microsoft.App.environments'
        properties: {
          serviceName: 'Microsoft.App/environments'
        }
      }
    ]
    privateEndpointNetworkPolicies: 'Disabled' // Required for CAE internal environment
    privateLinkServiceNetworkPolicies: 'Enabled' 
  }
  {
    name: 'snet-aistudio-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 2) // 10.X.2.0/24
  }
  {
    name: 'snet-gpuvms-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 3) // 10.X.3.0/24
  }
  {
    name: 'snet-privateendpoints-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 4) // 10.X.4.0/24
    privateEndpointNetworkPolicies: 'Enabled' // Or 'Disabled' if issues with NSG application on PE subnet
  }
  // Add other subnets like AppGateway, Bastion if needed
]

resource vnet 'Microsoft.Network/virtualNetworks@2022-07-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [for (subnet, i) in subnets: {
      name: subnet.name
      properties: {
        addressPrefix: subnet.addressPrefix
        delegations: contains(subnet, 'delegations') ? subnet.delegations : []
        privateEndpointNetworkPolicies: contains(subnet, 'privateEndpointNetworkPolicies') ? subnet.privateEndpointNetworkPolicies : 'Enabled'
        privateLinkServiceNetworkPolicies: contains(subnet, 'privateLinkServiceNetworkPolicies') ? subnet.privateLinkServiceNetworkPolicies : 'Enabled'
        // Add NSG association here if NSGs are created in this module
      }
    }]
  }
}

// Example NSG (repeat for other subnets)
resource nsgContainerApps 'Microsoft.Network/networkSecurityGroups@2022-07-01' = {
  name: 'nsg-containerapps-${environment}-${substring(location, 0, 4)}'
  location: location
  properties: {
    securityRules: [
      // Define rules here
    ]
  }
}

// Associate NSG to subnet (example)
resource subnetContainerAppsNsgAssociation 'Microsoft.Network/virtualNetworks/subnets@2022-07-01' = {
  parent: vnet
  name: subnets[0].name // Assuming first subnet is for Container Apps
  properties: {
    networkSecurityGroup: {
      id: nsgContainerApps.id
    }
  }
}


// Private DNS Zones (example for blob)
resource privateDnsZoneBlob 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global' // Private DNS zones are global
  properties: {}
}

resource vnetLinkBlob 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZoneBlob
  name: '${vnetName}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}
// ... Repeat for other required Private DNS Zones ...

output vnetId string = vnet.id
output vnetName string = vnet.name
output containerAppsSubnetId string = vnet.properties.subnets[0].id // Adjust index as per your subnets array
output privateEndpointSubnetId string = vnet.properties.subnets[3].id // Adjust index
// Output other subnet IDs as needed
```

### `modules/aistudio.bicep`

```bicep
// modules/aistudio.bicep
param aiStudioName string // This will be the Azure Machine Learning Workspace name
param location string
param environment string

param keyVaultId string
param privateEndpointSubnetId string // For AI Studio's dependent services private endpoints

// AI Studio requires: Storage, Key Vault, App Insights, Container Registry (optional but recommended)
// For simplicity, we'll show AML Workspace which is the core of AI Studio

var storageAccountName = 'st${replace(aiStudioName, '-', '')}${uniqueString(resourceGroup().id)}' // Create globally unique name
var appInsightsName = 'appi-${aiStudioName}'
var mlWorkspaceName = aiStudioName // AI Studio Project is an AML Workspace

resource storage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    // Configure for private access if possible, or use service endpoints initially
    publicNetworkAccess: 'Disabled' // Aim for this with Private Endpoints
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource mlWorkspace 'Microsoft.MachineLearningServices/workspaces@2023-04-01' = {
  name: mlWorkspaceName
  location: location
  sku: {
    name: 'Standard' // Or 'Enterprise' if features are needed; 'Basic' also exists
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: mlWorkspaceName
    storageAccount: storage.id
    keyVault: keyVaultId
    applicationInsights: appInsights.id
    // hbiWorkspace: false // High Business Impact workspace
    publicNetworkAccess: 'Disabled' // Important for private setup
  }
}

// Private Endpoints for AML Workspace and its storage
// This is a simplified example; AML has multiple sub-resources for PEs (api, notebook, etc.)
resource peStorageBlob 'Microsoft.Network/privateEndpoints@2022-07-01' = {
  name: 'pe-${storageAccountName}-blob'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${storageAccountName}-blob-connection'
        properties: {
          privateLinkServiceId: storage.id
          groupIds: ['blob'] // For blob storage
        }
      }
    ]
  }
}
// ... other PEs for file, queue, table for storage, and for AML workspace itself ...

output mlWorkspaceId string = mlWorkspace.id
output mlWorkspaceName string = mlWorkspace.name
```

### `modules/containerapps_env.bicep`

```bicep
// modules/containerapps_env.bicep
param caeName string
param location string
param environment string

param appSubnetId string // Subnet for the apps
// param controlPlaneSubnetId string // Subnet for the control plane (if using internal environment) - now often same as appSubnetId
param logAnalyticsWorkspaceId string // Optional, but recommended
param keyVaultId string // For secrets

// If Log Analytics Workspace is not passed, create a new one or decide on a naming convention
resource laWorkspace 'Microsoft.OperationalInsights/workspaces@2022-10-01' = if (empty(logAnalyticsWorkspaceId)) {
  name: 'la-${caeName}-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

var actualLogAnalyticsId = !empty(logAnalyticsWorkspaceId) ? logAnalyticsWorkspaceId : laWorkspace.id

resource cae 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: caeName
  location: location
  sku: {
    name: 'Consumption' // Or 'Premium' for workload profiles environment
  }
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: reference(actualLogAnalyticsId, '2022-10-01').customerId
        sharedKey: listKeys(actualLogAnalyticsId, '2022-10-01').primarySharedKey
      }
    }
    vnetConfiguration: {
      internal: true // True for VNet integrated environment, requires specific subnet config
      infrastructureSubnetId: appSubnetId // For internal environments, this is where infra is provisioned
      // runtimeSubnetId: appSubnetId // For workload profiles based environment
    }
    zoneRedundant: false // Consider true for production in regions that support it
  }
}

// Example of a Container App within the environment
/*
resource myContainerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'myapp-${environment}'
  location: location
  properties: {
    managedEnvironmentId: cae.id
    configuration: {
      // Secrets can come from Key Vault via reference
      // secrets: [
      //   {
      //     name: 'mysecret'
      //     keyVaultUrl: 'kvreference_to_secret_uri'
      //     identity: 'system' // Or user-assigned managed identity
      //   }
      // ]
      ingress: {
        external: false // Internal only
        targetPort: 80 
        transport: 'http'
        // allowInsecure: false // if using http
      }
      // dapr: { enabled: true, appId: 'myapp' ... }
      // registries: [ ... ]
    }
    template: {
      containers: [
        {
          image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
          name: 'simple-hello-world'
          resources: {
            cpu: json('0.25')
            memory: '0.5Gi'
          }
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 3
      }
    }
  }
}
*/

output caeId string = cae.id
output caeName string = cae.name
output caeDefaultDomain string = cae.properties.defaultDomain
```

### `modules/keyvault.bicep`

```bicep
// modules/keyvault.bicep
param kvName string
param location string
param environment string

param vnetId string // Not directly used for KV creation, but for context or if NSG rules depend on it
param privateEndpointSubnetId string

// Recommended: Enable RBAC for permissions
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: kvName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'Standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true // Recommended over access policies
    // softDeleteRetentionInDays: 90
    // enableSoftDelete: true
    publicNetworkAccess: 'Disabled' // Secure with Private Endpoint
  }
}

resource peKeyVault 'Microsoft.Network/privateEndpoints@2022-07-01' = {
  name: 'pe-${kvName}'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${kvName}-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault'] // Group ID for Key Vault
        }
      }
    ]
  }
}

// Example: Granting an identity access to the Key Vault (e.g., an App Service or VM Managed Identity)
// param principalIdToGrantAccess string
// resource kvSecretUserRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
//   scope: keyVault
//   name: guid(keyVault.id, principalIdToGrantAccess, 'kvSecretUser')
//   properties: {
//     roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
//     principalId: principalIdToGrantAccess
//     principalType: 'ServicePrincipal' // Or 'User', 'Group'
//   }
// }


output keyVaultId string = keyVault.id
output keyVaultUri string = keyVault.properties.vaultUri
```

This structure provides a scalable and organized way to deploy the Azure foundation using Bicep. Each module focuses on a specific area, making it easier to manage and update.
Remember to replace placeholders like `<env>` and adjust CIDR blocks, names, and SKU selections according to your specific project requirements.
