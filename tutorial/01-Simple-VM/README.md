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
parameters:
variables:
resources:
outputs:

```




```
