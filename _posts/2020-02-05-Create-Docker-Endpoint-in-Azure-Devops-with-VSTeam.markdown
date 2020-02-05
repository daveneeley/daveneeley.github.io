---
layout: post
title:  "Create Docker Endpoint in Azure Devops with VSTeam"
date:   2020-02-05 10:30:00
categories: code
---

Yesterday I wrote [Create a Custom Endpoint in Azure Devops with VSTeam]({% post_url 2020-02-04-Create-Custom-Endpoint-in-Azure-Devops-with-VSTeam %}). You'll need some of the scripts from that project for today's post.

Azure Container Registry (ACR) is a powerful tool within Azure. One way to take advantage of ACR in Azure Pipelines is to create a service endpoint in an Azure DevOps project and use it like a container registry, where you build and run container images on an agent in a pool. But ACR also has the concept of tasks, where you build and run container images within ACR itself. I came across this second option when I wanted to build images for the Raspberry Pi, which does not have an agent pool in Azure DevOps.

Here's my script for creating an Docker Registry connection backed by ACR. Save this file as `azure/vsteamregistryendpoint.ps1`, in the same project as the scripts from yesterday. The `azure/vsteamendpoint.ps1` script is a dependency of this script. My ACR exists once in one resource group, and is shared by other projects in Azure DevOps (and their corresponding resource groups). The heavy lifting is done by `azure/vsteamendpoint.ps1`, in this script we're defining the scopes and properties of the endpoint that are different from the base script.

``` Powershell
<#
    .Synopsis
    Create an Azure App Registration, Azure Devops Service Endpoint for an ACR-based Docker Registry

    .Parameter $account
    The Azure DevOps account name (also called the project collection name in some docs)

    .Parameter $project
    The Azure DevOps project name within the project collection

    .Parameter $resourceGroupName
    The name of the Azure Resource Group to which the endpoint will be scoped

    .Parameter $registryResourceGroupName
    The name of the Azure Resource Group that is linked to the azure container registry

    .Parameter $registryShortName
    The common name of the azure container registry

    .Parameter $token
    Override the Azure DevOps Bearer token with a personal access token, useful when running outside an Azure Pipeline build

#>
param (
  $account,
  $project,
  $resourceGroupName,
  $registryResourceGroupName,
  $registryShortName,
  $token = $env:SYSTEM_ACCESSTOKEN
)

#################### Functions
function New-VSTeamInstance {
  param (
    $account,
    $project
  )
  if (!(Get-Command -module VSTeam)) {
    write-host installing VSTeam
    Install-Module VSTeam -Force
    Import-Module VSTeam -Force -NoClobber
  }
  $accountParams = @{account=$account;token=$token}
  if ($env:SYSTEM_ACCESSTOKEN -and $token -eq $env:SYSTEM_ACCESSTOKEN) {
    write-verbose "using bearer token"
    $accountParams.UseBearerToken = $true
  }
  Set-VSTeamAccount @accountParams
  Set-VSTeamDefaultProject $project
  $team = Get-VSTeamProject -Name $project
  if (!$team) {
    throw "Please add the project '$project' in azure devops first"
  }
}
#################### End Functions

#get the path to the endpoint creation script
$parentPath = Split-Path $script:MyInvocation.MyCommand.Path -parent
Set-Alias docker-endpoint (Join-Path $parentPath vsteamendpoint.ps1)
New-VSTeamInstance -account $account -project $project -token $token

#get some azure information
$azcontext = Get-AzContext

$docker_scope = "/subscriptions/$($azcontext.Subscription.Id)/resourceGroups/$registryResourceGroupName/providers/Microsoft.ContainerRegistry/registries/$registryShortName"
$docker_url = $registryShortName + ".azurecr.io"
#this is the AcrPush role: https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles
$docker_role = 'AcrPush' #"8311e382-0749-4cb8-b61a-304f252e45ec"

$projectInfo = Get-VsTeamProject $project

$data = @{ 
  data = @{ 
    registryId   = $docker_scope
    registryType = "ACR"
  }
  authorization = @{
    parameters = @{
      scope = $docker_scope
      role  = $docker_role
      loginServer = $docker_url
      authenticationType = $null
    }
  }
  description = ""
  url   = "https://$docker_url"
  owner = "library"
  serviceEndpointProjectReferences=@(@{  
    name = "acr-$project"
    description = ""
    projectReference = @{
      name = $project
      id = $projectInfo.Id
    }
  })
}

#creates the "service connection" in the azure pipeline project
docker-endpoint -account $account -project $project -resourceGroupName $resourceGroupName -endpointType dockerregistry -endpointName "acr-$project" -data $data -token $token -scope $docker_scope -role $docker_role
```

The tasks feature of ACR is available in the Azure CLI (and probably other places too). The Azure CLI task in Azure DevOps uses the standard Azure Resource Manager service connection. Use the script from [yesterday's post]({% post_url 2020-02-04-Create-Custom-Endpoint-in-Azure-Devops-with-VSTeam %}) to make the endpoint in your Azure DevOps project.

This script adds permission to your existing app registration (and corresponding service principal) to run ACR Tasks. Save it as `azure/acrtaskrunnerrole.ps1` in the same project as the other scripts.

``` Powershell
<#
    .Synopsis 
    Creates the 'AcrTaskRunner' role and assigns it to a service principal

    .Parameter $account
    The Azure DevOps account name (also called the project collection name in some docs)

    .Parameter $project
    The Azure DevOps project name within the project collection

    .Parameter $resourceGroupName
    The name of the Azure Resource Group to which the endpoint will be scoped

    .Parameter $registryResourceGroupName
    The name of the Azure Resource Group that is linked to the azure container registry

    .Parameter $registryShortName
    The common name of the azure container registry
#>
param (
  $account,
  $project,
  $resourceGroupName,
  $registryResourceGroupName,
  $registryShortName
)

#################### Functions
function makeRoleAssignment {
  param (
    $roleName,
    $sp,
    $registry
  )

  $role = $sp | Get-AzRoleAssignment -RoleDefinitionName $roleName | ?{$_.scope -eq $registry.id -and $_.displayname -eq $sp.displayname}
  if (!$role) {
    if ($sp -and $registry -and !$role) {
      New-AzRoleAssignment -ObjectId $sp.Id -RoleDefinitionName $roleName -Scope $registry.id
    } else {
      throw "Could not assign $roleName on $($registry.name) to service principal $($sp.displayname)"
    }
  }
}

$azcontext = Get-AzContext
#need the ability to run tasks, which is a feature of ACR registries
$taskRoleName = 'AcrTaskRunner'
$template = Get-AzRoleDefinition -Name $taskRoleName
if (!$template) {
  #start from an existing role definition
  $template = Get-AzRoleDefinition -Name AcrPush
  $template.Id = $null
  $template.Name = $taskRoleName
}
#assign and create a custom role for acr build access 
#https://github.com/Azure/acr/issues/174
$builderAction = 'Microsoft.ContainerRegistry/registries/scheduleRun/action'
$listAction = 'Microsoft.ContainerRegistry/registries/listBuildSourceUploadUrl/action'
$listLogSasAction = 'Microsoft.ContainerRegistry/registries/runs/listLogSasUrl/action'
$readAction = 'Microsoft.ContainerRegistry/registries/read'
#Follow example from `get-help New-AzRoleDefinition`
$template.Description = 'Grants access to acr tasks such as show, build, and run'
$template.Actions.RemoveRange(0,$template.Actions.Count)
$template.Actions.Add($builderAction)
$template.Actions.Add($listAction)
$template.Actions.Add($listLogSasAction)
$template.Actions.Add($readAction)
$template.AssignableScopes.Clear()
$subscriptionScope = "/subscriptions/" + $azcontext.Subscription.Id
$template.AssignableScopes.Add($subscriptionScope)
if ($template.Id) {
  Set-AzRoleDefinition -Role $template
} else {
  New-AzRoleDefinition -Role $template
}

#allow the azurerm service connection to pull and push to the registry with azure cli or azure powershell
#example here: https://github.com/Azure/azure-docs-powershell-samples/blob/master/container-registry/service-principal-assign-role/service-principal-assign-role.ps1
$registry = Get-AzContainerRegistry -ResourceGroupName $registryResourceGroupName -Name $registryShortName
$spname = "$account-azurerm-$project-application"
$sp = Get-AzAdServicePrincipal -DisplayName $spname
makeRoleAssignment -roleName AcrPush -sp $sp -registry $registry
makeRoleAssignment -roleName $taskRoleName -sp $sp -registry $registry
```

For my use cases, these two scripts are additional steps that my bootstrap project will do for each additional Azure DevOps project I create within my organization. I've added them to my template at `templates/resource-group.yml`.

<!-- {% raw %} -->
``` yaml
parameters:
  connection: '' #name of arm connection in bootstrap project
  project: '' #name of project within azure devops collection
  region: '' #region where resources will be located
  account: '' #name of azure devops collection (dev.azure.com/<account>/)
  docker_registry: '' #short name of the docker registry docker_registry(.azurecr.io>
  docker_rg: '' #resource group where the docker registry is located

jobs:
- job:
  steps:
  - task: AzurePowershell@4
    displayName: 'arm connection'
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
      VERBOSEPREFERENCE: Continue
    inputs:
      azureSubscription: ${{ parameters.connection }}
      scriptType: FilePath
      scriptPath: azure/vsteamendpoint.ps1
      scriptArguments:
        -account ${{ parameters.account }}
        -project ${{ parameters.project }}
        -resourceGroupName ${{ parameters.project }}-${{ parameters.region }}
        -endpointType azurerm
      azurePowershellVersion: latestVersion
  - task: AzurePowershell@4
    displayName: 'acr connection'
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
    inputs:
      azureSubscription: ${{ parameters.connection }}
      scriptType: FilePath
      scriptPath: azure/vsteamregistryendpoint.ps1
      scriptArguments:
        -account ${{ parameters.account }}
        -project ${{ parameters.project }}
        -resourceGroupName ${{ parameters.project }}-${{ parameters.region }}
        -registryResourceGroupName ${{ parameters.docker_rg }}
        -registryShortName ${{ parameters.docker_registry }}
      azurePowershellVersion: latestVersion
  - task: AzurePowershell@4
    displayName: 'acr task role'
    env:
      SYSTEM_ACCESSTOKEN: $(System.AccessToken)
    inputs:
      azureSubscription: ${{ parameters.connection }}
      scriptType: FilePath
      scriptPath: azure/acrtaskrunnerrole.ps1
      scriptArguments:
        -account ${{ parameters.account }}
        -project ${{ parameters.project }}
        -resourceGroupName ${{ parameters.project }}-${{ parameters.region }}
        -registryResourceGroupName ${{ parameters.docker_rg }}
        -registryShortName ${{ parameters.docker_registry }}
      azurePowershellVersion: latestVersion
```
<!-- {% endraw %} -->

I've added two more parameters to the template above, the values for these need to be added to `azure-pipelines.yml` for each new project.

``` yaml
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: my_resource_group_and_region
  jobs:
  - template: templates/resource-group.yml
    parameters:
      connection: <put your bootstrap project's connection name here>
      project: <put the new project name here>
      region: <put the region where resources will be deployed here>
      account: <put the name of the azure devops account (project collection) here>
      docker_registry: <put the name of the ACR here>
      docker_rg: <put the name of the resource group where the ACR is located here>
```

I'll note again that setting the correct permissions on the bootstrap service connection is the most challenging part of infrastructure-as-code in Azure DevOps. For every article I read where a fine-grained permission was suggested, there were several more suggestions to grant the bootstrap connection administrator permissions and move on!