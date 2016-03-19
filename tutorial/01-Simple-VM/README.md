# Tutorial 1

## Single VM Resource Manager Deployment

The YAML templates in this directory describe a simple deployment of a
single VM into a resource group.

## Usage

Make sure your azure client is logged in, and configured for Azure Resource
Manager mode:

$ azure login
$ azure config mode arm

From the current directory, run `make` to create the JSON files, or
`make deploy` to create them and deploy them to Azure.

$ azure group create myresourcegroup westus
$ make
$ make deploy RG=myresourcegroup

## Overview

Deploying an Azure VM into a Resource Manager group actually results in a
number of infrastructure resources being created:

1. The Group itself
2. A local storage account for the group
2. A virtual network
3. A number of virtual subnets attached to that network
4. A number of virtual NICs attached to those subnets
5. Finally, a VM to which one of those NICs are attached

For small deployments, this is a lot of work. At scale, all this groundwork
really starts to pay dividends.

### Create the Resource Group

Creating the resource group is quick and easy:

```bash
$ azure group create rg-01 northeurope

```

*Hint*: Use `azure location list` to see the names of the available locations.

### Start your template

First order of the template is the metadata and skeleton layout.  A template
skeleton looks like this:

YAML:

```yaml

$schema: "$schema: http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#"
contentVersion: "1.0.0.0"
parameters: {}
variables: {}
resources: []
outputs: {}
```

JSON:

```json
"$schema": "$schema: http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
"contentVersion": "1.0.0.0",
"variables": {},
"parameters": {},
"resources": [],
"outputs": []
```

### Set your variables

For this template, there are a few values that need to be used over and over
again.  We're going to create them as variables:

```yaml
variables:
  location: "[resourceGroup().location]"
  apiVersion: "2015-06-15"
  adminUserName: "[parameters('adminUserName')]"
  adminPassword: "[parameters('adminPassword')]"
```

You'll notice that we're referencing the template function parameters() in
that last bit. No worries, we'll get to that in just a bit.

### Create your resources

#### Local Storage Account

Storage accounts are where the disks, file shares, and other stored data
of a resource group are kept.

Storage names have to be uniqiue across an account, so we'll be lazy and use
automatic name generation to take care of this.

First, we create a variable to store the name of the storageAccount:

```yaml
variables:
  newStorageAccountName: "[concat('storage', uniqueString(resourceGroup().id))]"
  storageAccountType: Standard_LRS
```

The storage account type has been set to "Standard_LRS".  The possible types
are:

* Standard_LRS
* Standard_ZRS
* Standard_GRS
* Standard_RAGRS
* Premium_LRS

For pricing, see the [Azure Storage Pricing documentation][storage_pricing].

This uses two new functions:

* `concat`: Like python's `string.join`
* `uniqueString()`: a hash made from the supplied parameter, in this case
  `resourceGroup().id`

The resulting name will be something like `storageabexxyxasdf`.

Second, we create the resource definition under `resources:` in our template:

```yaml
resources:
  - type: "Microsoft.Storage/storageAccounts"
    name: "[variables('newStorageAccountName')]"
    apiVersion: "[variables('apiVersion')]"
    location: "[variables('location')]"
    properties:
      accountType: "[variables('storageAccountType')]"
```

In theory, we could crack on now and deploy this template. Let's convert it to
JSON and give it a whirl.

Save the file as `simple-vm.yaml` and use the `compile.py` script to turn it
to JSON:

```bash
../../utils/compile.py simple-vm.yaml
Compiling simple-vm.yaml
Compiling base template:
- simple-vm.yaml -> simple-vm.json
OK
```
The resulting template looks like this:

```json

{
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "outputs": {},
  "variables": {
    "adminUserName": "[parameters('adminUserName')]",
    "newStorageAccountName": "[concat('storage-', uniqueString(resourceGroup().id",
    "location": "[resourceGroup().location]",
    "apiVersion": "2015-06-15",
    "adminPassword": "[parameters('adminPassword')]"
  },
  "$schema": "$schema: http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "resources": [
    {
      "location": "[variables('location')]",
      "type": "Microsoft.Storage/storageAccounts",
      "properties": {
        "accountType": "[variables('storageAccountType')]"
      },
      "apiVersion": "[variables('apiVersion')]",
      "name": "[variables('newStorageAccountName')]"
    }
  ]
}
```
Let's give that a deploy!

```bash
$ azure group create rg-01 northeurope
$ azure group deployment create -f simple-vm.json rg-01
```

Looking in the Azure portal, you should see something like this:

![Storage Deployment Succeeded][storage_succeeded]

### Define Parameters

So far, we're referencing two parameters:

* `adminUserName`
* `adminPassword`

Defining parameters is pretty straightforward:

```yaml

parameters:
  adminUserName:
    type: string
    metadata:
      description: User name for administrator
```


[storage_pricing]:https://azure.microsoft.com/en-us/pricing/details/storage/
[storage_succeeded]:https://dl.dropboxusercontent.com/u/12268528/ehwio-azure-templates/01_simple-vm/01_simple-vm_storage.png
