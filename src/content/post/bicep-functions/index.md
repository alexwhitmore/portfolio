---
title: 'Create User Defined Functions with Azure Bicep'
description: 'Use helper modules to assist with repetitive tasks like handling IP address formats, dates, or even naming conventions.'
publishDate: '11 August 2024'
coverImage:
  src: './bicep.webp'
  alt: 'A logo of Azure Bicep, which is a domain-specific language that aims to simplify the process of deploying resources to Azure.'
tags: ['bicep', 'arm', 'azure']
---

## What is Azure Bicep?

[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep) is a domain-specific language (DSL) that uses a declarative syntax to deploy Azure resources. It provides concise syntax, reliable type safety, and support for code reuse.

**Note: Bicep CLI version 0.26.X or higher is required to use this feature.**

## What are user-defined functions?

Different than the [standard Bicep functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions) that are available by default in your Bicep files, user-defined functions are custom made function created by you. They are useful for when you have complicated expressions used repeatedly in your Bicep files and you need to enforce consistent logic.

## Limitations

- The function can't access variables.
- The function can only use parameters that are defined in the function.
- The function can't use the reference function or any of the list functions.
- Parameters for the function can't have default values.

## Your first function

```bicep
// main.bicep
// user-defined functions structure
func <user-defined-function-name> (<argument-name> <data-type>, <argument-name> <data-type>, ...) <function-data-type> => <expression>

func myFirstFunction (message string) string => 'Your first message is ${message}'

output myFirstFunctionOutput string = myFirstFunction('Hello, World!')
```

The output:

| Function name   | Argument type | Output                               |
| :-------------- | :------------ | :----------------------------------- |
| myFirstFunction | String        | Your first message is: Hello, World! |

## Examples

Here's a practical example of a user-defined function that generates network configurations based on the deployment environment:

```bicep
// network-config.bicep

@description('Generates network configuration based on environment.')
func getNetworkConfig(env string) object {
  var baseConfig = {
    addressPrefix: '10.0.0.0/16'
    subnets: [
      {
        name: 'default'
        addressPrefix: '10.0.0.0/24'
      }
      {
        name: 'AzureBastionSubnet'
        addressPrefix: '10.0.1.0/27'
      }
    ]
  }

  return env == 'Production' ? union(baseConfig, {
    ddosProtection: true
    firewallRules: [
      {
        name: 'AllowCorporate'
        startIpAddress: '203.0.113.0'
        endIpAddress: '203.0.113.255'
      }
    ]
  }) : baseConfig
}
```

This function takes an environment parameter and returns a different network configuration based on whether it's a production environment or not.

Here's how you would use this function in your main Bicep file:

```bicep
// main.bicep

param environment string = 'Production'
var vnetConfig = getNetworkConfig(environment)

resource virtualNetwork 'Microsoft.Network/virtualNetworks@2021-03-01' = {
  name: 'myVNet'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetConfig.addressPrefix
      ]
    }
    subnets: vnetConfig.subnets
    ddosProtectionPlan: {
      id: vnetConfig.ddosProtection ? ddosProtectionPlan.id : null
    }
  }
}
```

The function's output varies depending on the environment:

```json
// Production Environment Output
{
  "addressPrefix": "10.0.0.0/16",
  "subnets": [
    {
      "name": "default",
      "addressPrefix": "10.0.0.0/24"
    },
    {
      "name": "AzureBastionSubnet",
      "addressPrefix": "10.0.1.0/27"
    }
  ],
  "ddosProtection": true,
  "firewallRules": [
    {
      "name": "AllowCorporate",
      "startIpAddress": "203.0.113.0",
      "endIpAddress": "203.0.113.255"
    }
  ]
}

// Non-Production Environment Output
{
  "addressPrefix": "10.0.0.0/16",
  "subnets": [
    {
      "name": "default",
      "addressPrefix": "10.0.0.0/24"
    },
    {
      "name": "AzureBastionSubnet",
      "addressPrefix": "10.0.1.0/27"
    }
  ]
}
```

This example demonstrates how user-defined functions can help manage environment-specific configurations efficiently, reducing code duplication and potential errors.

## What I Did Before Functions

Before the introduction of user-defined functions in Bicep, I relied on what I called [helper modules](https://www.whitmore.io/posts/bicep-helper-modules/). This approach involved creating a separate Bicep file as a module, outputting the result, and using it throughout my codebase. Here's a quick example:

```bicep
// helper.bicep

var whiteListedIps = [
  {
    Name: 'Rule1'
    CIDR: '192.0.2.0/24'
    StartIP: '192.0.2.0'
    EndIP: '192.0.2.255'
  }
]

var cosmosFormat = [for address in whiteListedIps: {
  ipAddressOrRange: address.CIDR
}]

output cosmosFormat array = cosmosFormat
```

The usage in the main file would look like this:

```bicep
// main.bicep

module ipRangeHelper 'helper.bicep' = {
  name: 'ip-ranges'
}

module cosmos '../cosmos.bicep' = {
  name: 'cosmosDeploy'
  params: {
    cosmosAccountName: cosmosAccountName
    ipRules: ipRangeHelper.outputs.cosmosFormat
  }
}
```

This approach served me well for years, and it still has its merits. Some benefits of using helper modules include:

- The ability to encapsulate complex logic and include resource definitions if needed.
- The flexibility to return multiple outputs of various types.

However, user-defined functions offer several advantages over custom modules:

- Faster execution, as functions are evaluated during template compilation rather than at deployment time.
- Easy parameterization and use within the same file, eliminating the need for a separate deployment scope.
- Ideal for simple, repetitive logic that doesn't involve resource creation.

While both approaches have their place, the introduction of user-defined functions has significantly streamlined many of our common tasks in Bicep.

## How I'm Using Functions Today

I've recently transitioned from custom helper modules to user-defined functions for all our helper logic. This change has been particularly beneficial for handling structured IP range inputs, which are required by many of our resources but often in different formats.

```bicep
// ip-range-helper.bicep

var whiteListedIps = [
  {
    Name: 'Rule1'
    CIDR: '192.0.2.0/24'
    StartIP: '192.0.2.0'
    EndIP: '192.0.2.255'
  }
]

@export()
func cosmosIpFormat() array => [for address in whiteListedIps: {
  ipAddressOrRange: address.CIDR
}]

@export()
func storageIpFormat() array => [for address in whiteListedIps: {
  value: address.CIDR
  action: 'Allow'
}]
```

Here's an example of how we use these functions:

```bicep
// main.bicep

import { cosmosIpFormat, storageIpFormat } from 'ip-range-helper.bicep'

module storageAccountModule '../storage-account.bicep' = {
  name: 'storageDeploy'
  params: {
    storageAccountName: storageAccountName
    networkAcls: {
      defaultAction: 'deny'
      ipRules: storageIpFormat()
    }
  }
}

module cosmos '../cosmos.bicep' = {
  name: 'cosmosDeploy'
  params: {
    cosmosAccountName: cosmosAccountName
    ipRules: cosmosIpFormat()
  }
}
```

The outputs from these functions are:

```json
// Storage Account output
[
  {
    "value": "192.0.2.0/24",
    "action": "Allow"
  }
]

// Cosmos DB output
[
  {
    "ipAddressOrRange": "192.0.2.0/24"
  }
]
```

As you can see, while the IP addresses are the same, each resource requires a different format with different parameter names. This variation in required formats makes it an ideal use case for helper functions, allowing us to maintain consistency and reduce errors in our deployments.

## Conclusion

User-defined functions in Bicep provide a powerful way to reduce code duplication and enforce consistent logic within your infrastructure-as-code. They offer the flexibility to create both simple and complex functions, allowing you to encapsulate logic as needed.

By using functions instead of custom modules, you can achieve the same benefits of code reuse and consistency while avoiding the additional overhead associated with module deployment. This approach makes them more efficient and easier to maintain.
