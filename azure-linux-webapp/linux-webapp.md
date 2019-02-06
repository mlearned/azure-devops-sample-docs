---
title: Deploy an Azure Linux App Service
description: Deploy an Azure Linux App Service from Azure Pipelines or TFS
services: vsts
ms.prod: devops
ms.technology: devops-cicd
ms.topic: conceptual
ms.manager: 
ms.assetid:
ms.custom: 
ms.author: 
author: blhoop
ms.date: 02/02/2019
monikerRange: '>= tfs-2017'
---

# Deploy an Azure Linux Web App

You can automate your Azure Web Applications deployments with Azure pipelines and Azure Resource Manager (ARM) templates.

## Prerequisites

- Azure Subscription
- Azure DevOps Account and Team Project
- Azure DevOps Service Connection


## Example

You can find the sample ARM template on GitHub:

````
https://github.com/Azure/azure-quickstart-templates/tree/master/101-webapp-basic-linux

````

!!! May need a sentence her to direct user to another page explaining how to use [Visual Studio Code](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-manager-quickstart-create-templates-use-visual-studio-code?tabs=CLI), [Visual Studio](https://docs.microsoft.com/en-us/azure/azure-resource-manager/vs-azure-tools-resource-groups-deployment-projects-create-deploy) 
to create an ARM template

You can always modify a template to meet your needs, but for this example we’ll use the ARM templates below.

```json

{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "webAppName": {
      "type": "string",
      "metadata": {
        "description": "Base name of the resource such as web app name and app service plan "
      },
      "minLength": 2
    },
    "sku":{
      "type": "string",
      "metadata": {
        "description": "The SKU of App Service Plan "
      }
    },
    "linuxFxVersion" : {
        "type": "string",
        "defaultValue" : "php|7.0",
        "metadata": {
          "description": "The Runtime stack of current web app"
        }
    },
    "location": {
      "type": "string",
      "metadata": {
        "description": "Location for all resources."
      }
    }
  },
  "variables": {
    "webAppPortalName": "[concat(parameters('webAppName'), '-webapp')]",
    "appServicePlanName": "[concat('AppServicePlan-', parameters('webAppName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2017-08-01",
      "kind": "linux",
      "name": "[variables('appServicePlanName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "sku": {
        "name": "[parameters('sku')]"
      },
      "properties": {
        "reserved": true
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2016-08-01",
      "kind": "app",
      "name": "[variables('webAppPortalName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
        "siteConfig": {
          "linuxFxVersion": "[parameters('linuxFxVersion')]"
        }
      },
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]"
      ]
    }
  ]
}
```

---
You can provide parameter values in the parameters.json file, or have these values be overwritten at runtime in the Azure DevOps pipeline, which we’ll see later in this article.

```json

{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "webAppName": {
      "value": "<Web Application Name>"
    },
    "sku": {
      "value": "<App Service Plan Pricing Tier>"
    },
    "linuxFxVersion": {
      "value": "<Runtime Stack>"
    },
    "location": {
      "value": "<Resource Group Location>"
    }
  }
}
```

---

## Setting up Azure DevOps Pipeline

To set up the build and release pipeline by selecting the appropriate tasks. In your Azure DevOps project, select Pipelines > Builds. Under +New, select New build pipeline. Now select where your code is located. If you select Azure Repository you will be provided with an auto generated YAML file that will represent your build. The YAML file can be modified to your build specifications. Another option is to use the visual designer and manually select the build tasks.
!!(would like to add the tab's at this point to allow user to switch between the Designer build task's and the YAML file version. )

# [YAML](#tab/yaml)

```yaml
resources:

- repo: self

pool:
  vmImage: Hosted VS2017
  demands: msbuild

steps:
- task: MSBuild@1
  displayName: 'Build solution ARMDeployments/basic-linux-webapp'
  inputs:
    solution: 'ARMDeployments/basic-linux-webapp'

- task: CopyFiles@2
  displayName: 'Copy Files to: $(build.artifactstagingdirectory)'
  inputs:
    SourceFolder: 'ARMDeployments/basic-linux-webapp'
    Contents: '**\*.json'
    TargetFolder: '$(build.artifactstagingdirectory)'
    CleanTargetFolder: true
    OverWrite: true

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact: drop'
```

# [Designer](#tab/designer)

![[Select Repository]](./_img/azure-pipeline-select-repository.png)

---

There are several options for selecting a repository source. For this example we’ll select Azure Repos Git and accept the defaults for the Team Project, Repository and Branch then select Continue.

![[Select Source]](./_img/select-repository-source.png)

On the Select template blade, at the top you'll see a link to start with an Empty job, click Empty Job.
On the Tasks page, here you will start selecting the build tasks needed.
Give the build task a descriptive name. To the right of Agent job 1 select the + sign to add a build step. **You will use the + sign every time you want to add a task. Tasks can be moved in order by dragging and dropping them in the correct sequence.**
Click on the build filter and select MSBuild for your first task. Click the ellipses to the right of Project and navigate to the project for the build. Use the default values for the rest of the settings.

![[MSBuild Task]](./_img/azure-pipeline-msbuild-task.png)

The next task to add will be Copy Files to. Leave the default for the Display name for now. Use the ellipses to locate your Source folder for the project. Enter `**\*.json` into the Contents text box. This will filter looking for only the two .json template files in your project folder. Set the Target Folder to `$(build.artifactstagingdirectory)`. Under Advanced, check the boxes for Clean Target Folder and Overwrite. Under Control Option, Enabled should be check by default and make sure Run this task is set to `Only when all previous tasks have succeeded`.

![[Copy Files to task]](./_img/azure-pipeline-copy-files-task.png)

---

The final task to add will be Publish Build Artifact. For the task settings ensure that Path to publish is set to `$(Build.ArtifactStagingDirectory)`, Artifact name is drop and Artifact publish location is `Azure Pipelines/TFS`. The rest of the settings you can take the defaults. When finished, click the Save and queue then select Save and queue again to run the build.

![[Publish Artifact task]](./_img/azure-pipeline-publish-artifact-task.png)

After the build has been queued, you should see a build number. Click the build number link to follow the build as it is running. Once the build finishes and has succeeded, you are now ready to create the release to deploy your web application to Azure.

![[Build Job Results]](./_img/azure-pipeline-build-job.png)

## Creating Pipeline Release



Release:
Azure Resource Group Deployment:
Azure subscription = <Azure Service Connection>
Action = Create or update resource group
Resource group = <Azure Resource group>
Location = <Azure Resource group location>
Template location = Linked artifact
Template = <Using the ellipses navigate to the drop folder and select the .json file>
Template parameters = <Using the ellipses navigate to the drop folder and select the parameters.json file>
Override template parameters = <Here you can set parameter values and override the values in the parameters file.>
Deployment mode = <Incremental>

---

