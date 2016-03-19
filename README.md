# ehwio-azure-templates
# Creating Azure Templates

This repository contains tests and research on creating Azure Resource
Manager templats.


## Understanding Templates

Azure deployment templates orchestrate Azure resources in a programmatic way.

The basic structure of a deployment is as follows:

* Resource Group
  * Public IP
  * VM Resources
    * VM Extensions, i.e. Chef
  * Network Resources
    * Subnets
    * Network Interfaces
  * Storage Account
    * VM disks
    * Shared Storage


### Considerations

* Public IPs should be used sparingly

Only machines that need to be publicly accessible should have a public IP assigned.  Otherwise,
there are load-balancers, VPN gateways, and other services that are more suited to being
publicly accessible.

* Public IP, Subnets, and NICs must all be explicitly created

Each machine's network cards need to be declared in the template, or they won't be connected.

* Dependencies

Dependencies are important for resources in Azure, but also in the Chef-managed infrastructure itself.
If a role relies on other roles being present first, list them as a dependency in the template.



Once a template has been created, it can deployed using either the portal or the command-line tool, [azure-cli][azure-cli].

In this document, both YAML and JSON formats are referenced.  Azure's CLI and Portal only support JSON format. YAML,
however, can be trivially converted to JSON, and is much more humane for end users.  This is why it is suggested as an
alternative authoring format.

Example: Converting YAML to JSON with Python

This example requires the python YAML library.

* On Ubuntu: `sudo apt-get install python-yaml`

* Mac OS X virtualenv: `pip install PyYAML`

```python

#!/usr/bin/python


# example script
# usage:
#   ./compile.py MyTemplate.yaml > MyTemplate.json

import sys
import yaml
import json

template = yaml.load(open(sys.argv[-1]))
template_json = json.dumps(template)

# print to stdout
print template_json

```

Example: Converting YAML to JSON with Ruby

```ruby

#!/usr/bin/ruby
# usage:
#   ./compile.rb MyTemplate.yaml > MyTemplate.json

require 'yaml'
require 'json'

filename = ARGV[0]
template = YAML.load_file(filename)

template_json = JSON.pretty_generate(template)
puts template_json

```

Template Components
===================

Templates consist of 4 main sections:

* parameters
* variables
* resources
* outputs

Parameters
----------

Parameters define which values can be passed in at run time.  The basic structure is like this:

YAML:

```yaml

parameters:
  adminUserName:
    type: string
  adminPassword:
    type: secureString


```

JSON

```json

  "parameters": {
    "adminUserName": {
      "type": "string"
    },
    "adminPassword": {
      "type": "secureString"
    },
  }

```

Parameter files are in a similar format.

YAML:

```yaml

parameters:
  location:
    value: westeurope
  adminUserName:
    value: demouser
  adminPassword:
    value: Arpanet1

```

JSON:

```

"parameters": {
  "adminUserName": {
    "value": "demouser"
  },
  "adminPassword": {
    "value": "Arpanet1"
  },
}

```


Storage Account
===============

[Schema][storage_schema]

type:  "Microsoft.Storage/storageAccounts"


### location

Required.

This is the Azure location where this IP address will be allocated.  You can use a template function to make it hang out with the rest of the resource group:

```yaml

location: "[resourceGroup().location]"

```

### apiVersion

Always required on everything. "2015-06-15" is a safe bet.

name: Needs to be Unique...

### properties

Storage Accounts have a child object, properties.  Properties is a mapping object with the following maps:



#### accountType: string

Microsoft.Storage/storageAccounts: The type of this account.


YAML:

```

resources:
  # set up the storage account
  - type: "Microsoft.Storage/storageAccounts"
    name: "[parameters('newStorageAccountName')]"
    apiVersion: "[variables('apiVersion')]"
    location: "[parameters('location')]"
    properties:
      accountType: "[variables('storageAccountType')]"

```


JSON:

```

  "resources": [
    {
      "location": "[parameters('location')]",
      "type": "Microsoft.Storage/storageAccounts",
      "properties": {
        "accountType": "[variables('storageAccountType')]"
      },
      "apiVersion": "[variables('apiVersion')]",
      "name": "[parameters('newStorageAccountName')]"
    },
```


Public IP Address
=================

Public IP addresses belong to a resource group, and can be assigned to a machine, load balancer, or other endpoints.

It uses the following values

### type

Required.

Must be `Microsoft.Network/publicIPAddresses`

### name

The name in the portal for this IP address.  This is *not* the FQDN.  This will be used to assign to vNIC instances.

### location

Required.

This is the Azure location where this IP address will be allocated.  You can use a template function to make it hang out with the rest of the resource group:

```

location: "[resourceGroup().location]"

```

### apiVersion

Always required on everything. "2015-06-15" is a safe bet.

### properties

Public IPs have a child object, properties.  Properties is a mapping object with the following maps:

#### dnsSettings

Contains the following:

domainNameLabel: Host named used for name resolution.	 This is just the hostname part. The domain will be appended by Azure.

#### publicIPAllocationMethod

Required.

Can be either "static" or "dynamic".  

#### idleTimeoutInMinutes

Optional.

Value between 4 and 30


### Examples

YAML:

```

  - type: Microsoft.Network/publicIPAddresses
    name: "[variables('publicIPAddressName')]"
    apiVersion: "[variables('apiVersion')]"
    location: "[variables('location')]"
    properties:
      publicIPAllocationMethod: "[variables('publicIPAddressType')]"
      dnsSettings:
        domainNameLabel: "[toLower(concat(resourceGroup().name, '-', uniqueString(resourceGroup().id)))]"

```

JSON:

```

    {
      "name": "[variables('publicIPAddressName')]",
      "type": "Microsoft.Network/publicIPAddresses",
      "properties": {
        "publicIPAllocationMethod": "[variables('publicIPAddressType')]",
        "dnsSettings": {
          "domainNameLabel": ""[toLower(concat(resourceGroup().name, '-', uniqueString(resourceGroup().id)))]""
        }
      },
      "apiVersion": "[variables('apiVersion')]",
      "location": "[variables('location')]"
    },

```

##

## Azure VM

Create Machine with One NIC, One Public IP, and One Hard Drive

A template that creates a VM must also create the following:

- Virtual Network
- Network Interface
- Storage Account

If you need to get to it from the outside world, you also need to create a Public IP.

Example:

YAML:

```

  - type: "Microsoft.Compute/virtualMachines"
    name: "[variables('machineName')]"
    location: "[variables('location')]"
    apiVersion: "[variables('apiVersion')]"
    # need to wait for storage and NIC to be available
    dependsOn:
      - "[concat('Microsoft.Network/networkInterfaces/', variables('nicName'))]"
      - "[concat('Microsoft.Storage/storageAccounts/', variables('newStorageAccountName'))]"
    properties:
      hardwareProfile:
        vmSize: "[variables('vmSize')]"
      osProfile:
        computerName: "[variables('machineName')]"
        adminUsername: "[variables('adminUsername')]"
        adminPassword: "[variables('adminPassword')]"
        linuxConfiguration:
          ssh:
            publicKeys: "[parameters('sshPublicKeys')]"

      storageProfile:
        imageReference:
          publisher: "[variables('imagePublisher')]"
          offer: "[variables('imageOffer')]"
          sku: "[variables('imageSKU')]"
          version: latest
        osDisk:
          name: osdisk
          vhd:
            uri: "[concat('http://',parameters('newStorageAccountName'), '.blob.core.windows.net/',variables('vmStorageAccountContainerName'), '/', variables('OSDiskName'),'.vhd')]"
          caching: ReadWrite
          createOption: FromImage
      networkProfile:
        networkInterfaces:
          - id: "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
```


JSON:

```json

    {
      "name": "[variables('machineName')]",
      "dependsOn": [
        "[concat('Microsoft.Network/networkInterfaces/', variables('nicName'))]",
        "[concat('Microsoft.Storage/storageAccounts/', variables('newStorageAccountName'))]"
      ],
      "type": "Microsoft.Compute/virtualMachines",
      "properties": {
        "hardwareProfile": {
          "vmSize": "[variables('vmSize')]"
        },
        "storageProfile": {
          "imageReference": {
            "sku": "[variables('imageSKU')]",
            "publisher": "[variables('imagePublisher')]",
            "version": "latest",
            "offer": "[variables('imageOffer')]"
          },
          "osDisk": {
            "caching": "ReadWrite",
            "vhd": {
              "uri": "[concat('http://',parameters('newStorageAccountName'), '.blob.core.windows.net/',variables('vmStorageAccountContainerName'), '/', variables('OSDiskName'),'.vhd')]"
            },
            "createOption": "FromImage",
            "name": "osdisk"
          }
        },
        "osProfile": {
          "adminUsername": "[variables('adminUsername')]",
          "computerName": "[variables('machineName')]",
          "linuxConfiguration": {
            "ssh": {
              "publicKeys": "[parameters('sshPublicKeys')]"
            }
          },
          "adminPassword": "[variables('adminPassword')]"
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
            }
          ]
        }
      },
      "apiVersion": "[variables('apiVersion')]",
      "location": "[variables('location')]"
    }
  ]
}

```

## DBaaS Databases

type: SuccessBricks.ClearDB/database
name:
location:
plan:
  name: Plan Name (Titan, Mercury, etc.)
properties:
  hostname:
  name: db_name



Example Resource Output:

    {
      "id": "/subscriptions/06cde865-d643-4f75-86c5-e160c8b4e53e/resourceGroups/td-store-westeurope-1/providers/SuccessBricks.ClearDB/databases/test_db",
      "name": "test_db",
      "type": "SuccessBricks.ClearDB/database",
      "location": "West Europe",
      "plan": {
        "name": "Titan"
      },
      "properties": {
        "hostname": "eu-cdbr-azure-west-d.cloudapp.net",
        "name": "test_db",
        "id": "B69A39D0A082D0432A6EB1250E695B7D",
        "size_mb": 0,
        "max_size_mb": "250",
        "port": 3306,
        "status": {
          "name": "Healthy",
          "message": "Database is healthy and ready for use.",
          "level": "Info"
        }
      },
      "tags": {
        "DATABASE_ALIAS": "test_db"
      }
    }





[azure-cli]:https://azure.microsoft.com/en-us/documentation/articles/xplat-cli-install/
[storage_schema]:https://github.com/Azure/azure-resource-manager-schemas/blob/master/schemas/2015-08-01/Microsoft.Storage.json
