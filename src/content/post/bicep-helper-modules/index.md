---
title: "Create Helper Modules with Azure Bicep"
description: "Use helper modules to assist with repetitive tasks like handling IP address formats, dates, or even naming conventions."
publishDate: "16 June 2024"
coverImage:
  src: "./bicep.webp"
  alt: "A logo of Azure Bicep, which is a domain-specific language that aims to simplify the process of deploying resources to Azure."
tags: ["bicep", "arm", "azure"]
---

## What is Azure Bicep?

[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep) is a domain-specific language (DSL) that uses a declarative syntax to deploy Azure resources. It provides concise syntax, reliable type safety, and support for code reuse.

```bicep
// storage-account.bicep

@description('Optional. The location of the resource to be deployed in to.')
param location string = resourceGroup().location

@description('Required. The name of the Storage Account resource.')
param storageAccountName string

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-06-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
  }
}
```

A Bicep resource used for deploying a Standard Azure Storage Account.

---

## What are Bicep modules?

Modules are Bicep files that are deployed from another Bicep file which improve the readability of your code by encapsulating complex details of your deployment. Modules also unlock the ability to reuse code for different deployments. To learn more, you can [read the documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules).

```bicep
// main.bicep

var storageAccountName = 'toylaunch${uniqueString(resourceGroup().id)}'

module storageAccountModule '../storage-account.bicep' = {
  name: 'storageDeploy'
  params: {
    storageAccountName: storageAccountName
  }
}
```

Basic syntax for defining a Storage Account module - the above snippet is directly consuming the resource from `storage-account.bicep`.

---

## Why should I use modules?

Bicep modules will improve readability, reusability, and flatten your files. If this wasn't reason enough, you can create _helper modules_. A helper module is a Bicep module that does not deploy any Azure resources, but it can assist you with repetitive tasks where you may need formatted values.

This is helpful when you need certain values throughout your code base. Some examples are IP addresses, dates, and resource naming conventions. These values require a certain format and are frequently and as your code base grows, the more places you need to use them.

## Problem

Let's take IP addresses as an example. IP addresses require a certain format which is not always consistent between resources, which creates a challenge to manage your infrastructure. An incorrect format can result in deployment errors or connectivity issues, causing potential downtime. For example:

```bicep
module storageAccountModule '../storage-account.bicep' = {
  name: 'storageDeploy'
  params: {
    storageAccountName: storageAccountName
    networkAcls: {
      defaultAction: 'deny'
      // ipRules: IPRule[]
      ipRules: [
        {
          action: 'Allow'
          value: '192.0.2.0/24'
        }
      ]
    }
  }
}
```

The `IPRule` arrays are made up of objects with two values inside: **action** and **value**. Action is the action of the IP rule which has a type of `'Allow' | 'Deny'`, and value specifies the IP or IP range in [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) format.

The previous IP address configuration is of course valid for storage accounts, but this wouldn't work for something like CosmosDB.

```bicep
// main.bicep

module cosmos '../cosmos.bicep' = {
  name: 'cosmosDeploy'
  params: {
    cosmosAccountName: cosmosAccountName
    // ipRules: IPRule[]
    ipRules: [
      {
        ipAddressOrRange: '192.0.2.0/24'
      }
    ]
  }
}
```

Within `IPRule` only `ipAddressOrRange` is valid.

## Solution

To avoid misconfigurations or a large variable containing all IP addresses at the top of all your Bicep files, you can make use of a helper module.

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

var cosmosFormat = [for address in whiteListedIps: {
  ipAddressOrRange: address.CIDR
}]

var storageFormat = [for address in whiteListedIps: {
  value: address.CIDR
  action: 'Allow'
}]

output cosmosFormat array = cosmosFormat
output storageFormat array = storageFormat
```

Now to consume this module:

```bicep
// main.bicep

module ipRangeHelper '../ip-range-helper.bicep' = {
  name: 'ip-ranges'
}

module storageAccountModule '../storage-account.bicep' = {
  name: 'storageDeploy'
  params: {
    storageAccountName: storageAccountName
    networkAcls: {
      defaultAction: 'deny'
      ipRules: ipRangeHelper.outputs.storageFormat
    }
  }
}

module cosmos '../cosmos.bicep' = {
  name: 'cosmosDeploy'
  params: {
    cosmosAccountName: cosmosAccountName
    ipRules: ipRangeHelper.outputs.cosmosFormat
  }
}
```

---

## Summary

Bicep modules are handy for improving readability and reusability in your code, but are also a powerful tool for repetitive tasks and enforcing consistency in your infrastructure. These helpers allow you to define a consistent formatting one time which can then be used everywhere.
