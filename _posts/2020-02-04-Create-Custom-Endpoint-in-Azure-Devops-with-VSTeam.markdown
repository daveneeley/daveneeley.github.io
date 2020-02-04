---
layout: post
title:  "Create Custom Endpoint in Azure Devops with VSTeam"
date:   2020-02-04 12:00:00
categories: code
---

I'm very happy to have found the [vsteam](https://github.com/DarqueWarrior/vsteam) Powershell module on GitHub. At work we have a custom application to provision resource groups in Azure, but for my home projects it's powershell all the way. I've seen some other modules, but vsteam has a very clean implementation. It's great stuff.

I've been bootstrapping my home Azure DevOps projects and their Azure resources from within Azure DevOps. That is, I use one Azure DevOps project and infrastructure as codee to create more Azure DevOps projects within my account. It's a work in progress, but it has helped me to better understand the permissions model of Azure DevOps and Azure too. This can be challenging when trying to enforce the principle of least priviledge.

Here's my script for creating an Azure Resource Manager (arm or azurerm) connection that is scoped to a particular resource group. Save this file as `azure/vsteamendpoint.ps1`.

``` Powershell
<#
    .Synopsis Create an Azure App Registration and Azure Devops Service Endpoint

    .Parameter $account
    The Azure DevOps account name (also called the project collection name in some docs)

    .Parameter $project
    The Azure DevOps project name within the project collection

    .Parameter $resourceGroupName
    The name of the Azure Resource Group to which the endpoint will be scoped

    .Parameter $endpointType
    The string 'id' of the new endpoint's type in Azure Devops, see the list with Get-VsTeamEndpointTypes

    .Parameter $scope
    A custom Azure Graph scope if overriding the default resource group scope

    .Parameter $role
    A custom role given to the service principle during creation only, default is Contributor

    .Parameter $data
    A hashtable of additional Azure Devops Service Endpoint settings that will be merged in with the default set of endpoint settings. Defaults are overridden by passed in data

    .Parameter $appName
    Override the generated but readable name of the Azure App Registration as seen in Azure Portal

    .Parameter $endpointName
    Override the generated but readable name of the endpoint as seen in Azure Devops

    .Parameter $spName
    Override the generated but readable name of the Service Principal created with the App Regisration as seen in Azure Portal

    .Parameter $token
    Override the Azure DevOps Bearer token with a personal access token, useful when running outside an Azure Pipeline build

    .Parameter $ignoreEndPointCreationErrors
    Assume the endpoint was created even if it told you the build service can't access it later
    The Azure Devops REST API will not display endpoints created by a build service account when the list is requested by the build service account

#>
param (
  $account,
  $project,
  $resourceGroupName,
  $endpointType,
  $scope,
  $role = "Contributor",
  $data = @{},
  $appname = "$account-$endpointType-$project-application",
  $endpointName = "$endpointType-$project",
  $spname = "$appname-sp",
  $token = $env:SYSTEM_ACCESSTOKEN,
  $ignoreEndPointCreationErrors = $true
)

###################### Functions

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

function New-VSTeamServiceEndpointSettings {
  param (
      $azContext,
      $app,
      $pass,
      $scope,
      $data
  )
  $settings = @{
    data = @{
      SubscriptionId = $azContext.Subscription.Id
      SubscriptionName = $azContext.Subscription.Name
      environment = $azContext.Environment.Name
    }
    authorization = @{
      parameters = @{
        tenantId = $azcontext.Tenant.Id
        servicePrincipalId = $app.ApplicationId
        servicePrincipalKey = $pass
        authenticationType = "spnKey"
        scope = $scope
      }
      scheme = "ServicePrincipal"
    }
    isShared = $true
  }
  $settings, $data | Merge-Object -Strategy Override
} 

#from https://gist.github.com/marcgeld/4891bbb6e72d7fdb577920a6420c1dfb
function Get-RandomAlphanumericString {
  [CmdletBinding()]
  Param (
    [int] $length = 8
  )
  return ( -join ((0x30..0x39) + ( 0x41..0x5A) + ( 0x61..0x7A) | Get-Random -Count $length | % {[char]$_}) )
}

###################

#used for Merge-Object
if (!(Get-Command -module Functional)) {
  Install-Module Functional -Force
  Import-Module Functional -Force
}
#used for azure commands
if (!(Get-Command -module 'Az.*')) {
  Install-Module Az -Force
  Import-Module Az -Force
}
New-VSTeamInstance -account $account -project $project -token $token
$team = Get-VSTeamProject -Name $project
if (!$team) {
  throw "Please add the project '$project' in azure devops first"
}
$azcontext = Get-AzContext
$app = Get-AzAdApplication -DisplayName $appname
if (!$app) {
  write-host creating application $appname
  $app = New-AzADApplication -DisplayName $appname -HomePage "https://VisualStudio/SPN" -IdentifierUris "https://Visualstudio/SPN/$appname"
}
if (!$scope) {
  $scope = "/subscriptions/$($azcontext.Subscription.Id)/resourcegroups/$resourceGroupName"
}
$sp = $app | Get-AzAdServicePrincipal
if (!$sp) {
  write-host creating service principal $spname
  $sp = New-AzAdServicePrincipal -ApplicationId $app.ApplicationId -Displayname $spname -Scope $scope -Role $role
}
$ep = Get-VSTeamServiceEndpoint
$ep = $ep | Where-Object {$_.name -eq $endpointName}
if (!$ep) {
  write-host creating endpoint $endpointName
  # if we don't already have a connection, we need to create a new password for the app registration
  $passString = Get-RandomAlphanumericString -length 22
  $securepass = ConvertTo-SecureString $passString -AsPlainText -Force
  $void = $app | New-AzADAppCredential -Password $securepass -StartDate $(Get-Date) -EndDate $((Get-Date).AddDays(365))

  $epSettings = New-VSTeamServiceEndpointSettings -azcontext $azcontext -app $app -pass $passString -scope $scope -data $data

  try
  {
    $ep = Add-VSTeamServiceEndpoint -endpointName $endpointName -endpointType $endpointType -object $epSettings
  } catch {
  if ($_ -notmatch "already exists") {
    throw
  }
  write-warning $_
  write-output $($_ | get-member)
  $ep = Get-VSTeamServiceEndpoint | Where-Object {$_.name -eq $endpointName}
  if (!$ep -and !$ignoreEndPointCreationErrors) {
      #See note in parameter description to understand why this is necessary
      throw "could not create endpoint"
  }
}
```

Save the script below as `templates/resource-group.yml`. Since I use this for multiple projects, it's in a yaml template file. I've left out the step that creates the resource group.

<!-- {% raw %} -->
``` yaml
parameters:
  connection: '' #name of arm connection in bootstrap project
  project: '' #name of project within azure devops collection
  region: '' #region where resources will be located
  account: '' #name of azure devops collection (dev.azure.com/<account>/)

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
```
<!-- {% endraw %} -->

Save this file as `azure-pipelines.yml`. After you push this code, start a new pipeline from an existing script in Azure Devops and use this file.

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
```

I've read various github issues, azure devops uservoice tickets, API and Graph docs to find the right set of permissions. I think I've got it mostly right. I've been keeping these notes in the README of my project. I've tried to update the readme every time I manually tweaked something in Azure Devops or Azure portal. Some of the steps at the end of the list are hack attempts at the issue of the azure devops build service account being unable to view endpoints that it created. That's still unresolved.

```
Login to Azure Devops
Create Subscription in Azure Portal
Add Azure Active Directory to Azure Devops (org settings -> general -> azure active directory -> connect)
NOTE: This will invalidate all of your pre-existing personal access tokens...
Add bootstrap project in Azure Devops
Build a pipeline to get bootstrap build service created in Azure Devops
Add bootstrap build service to Project Collection Service Accounts in Azure Devops (it's vsts only)
Create my-bootstrap-service-connection in Azure Devops bootstrap project, it's an Azure Resource Manager connection
Find app registration for my-bootstrap-service-connection in azure portal
Rename to find easier later my-bootstrap-app
Grant 'Application Manager' to my-bootstrap-app in azure portal
https://stackoverflow.com/q/56486068/9660

#notes get fuzzier at this point, use at your own risk!
Grant 'Application.ReadWrite.OwnedBy', 'Application.Read.All', 'Application.ReadWrite.All', 'Directory.Read.All', 'Directory.ReadWrite.All' to my-bootstrap-app in azure portal (directory -> select user -> api permissions -> add permission -> microsoft graph -> 'application permissions' -> search/select 'Application.ReadWrite.OwnedBy' -> Add Permissions
Directory.AccessAsUser.All is also required, according to help docs for New-AzAdApplication. This is found under 'delegated permissions' instead of 'application permissions' in the above sequence.
Different links say different things about what is and isn't required - It would be nice if it was only 'Application.ReadWrite.OwnedBy'...
https://github.com/Azure/azure-powershell/issues/3215
Approve admin permission grant
Grant 'User Access Administrator' to my-bootstrap-app in Subscriptions -> (subscription) -> Access control (IAM) in azure portal
very helpful: https://docs.microsoft.com/en-us/azure/role-based-access-control/rbac-and-directory-admin-roles
Add project collection build service to Project Collection Service Accounts in Azure Devops (it's vsts only)
https://stackoverflow.com/questions/52672413/using-system-accesstoken-to-create-a-service-endpoint
Grant Azure Active Directory (legacy) 'Application.ReadWrite.OwnedBy' api permission to my-bootstrap-app in azure portal
covers "Resource not found for the segment 'me'" https://github.com/Azure/azure-powershell/issues/3215
Grant Azure Active Directory (legacy) 'Directory.Read.All' api permission to my-bootstrap-app in azure portal
```