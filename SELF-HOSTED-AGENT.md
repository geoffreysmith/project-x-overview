# Self-hosted Azure Devops agent
To circumvent the diskspace or CPU performance limitations with the default Azure Devops Microsoft-hosted agents you can build your own self-hosted agent.
Below you find the steps required to build a self-hosted agent using an Azure VM and the Microsoft defined Azure Devops agent Packer definition.

## Build the agent image

To build a self-hosted Azure Devops agent use this repo:

Install AzureRM Powershell Modules:
```
PS> Install-Module AzureRM -AllowClobber
```


```

Generate the Azure Devops agent image:
```
PS> powershell -command "&{ . .\helpers\GenerateResourcesAndImage.ps1; GenerateResourcesAndImage}"
```
Warning this takes around 6 hours to complete.

## Create an Azure VM from the image
Create and start the VM using the generated Packer template:
```
PS> powershell -command "&{ . .\helpers\CreateAzureVMFromPackerTemplate.ps1; CreateAzureVMFromPackerTemplate}"
```

Optionally change the desired `vmSize` in `helpers\CreateAzureVMFromPackerTemplate.ps1` to get a pre-scaled image (instead of the default size).

## Install the Azure Devops agent
The Azure Devops agent has to be installed afterwards, e.g. using a remote desktop connection.

First enable RDP;
- add Network Security Group (NSG) in Azure (under the same Resource Group), 

```
az network nsg create -g sitecorebuildagent -n sitecorebuildagent
```
- open RDP: https://docs.microsoft.com/en-us/azure/virtual-machines/troubleshooting/troubleshoot-rdp-nsg-problem
```
az network nsg rule create -g sitecorebuildagent --nsg-name sitecorebuildagent -n Port_3389 --priority 300 --source-address-prefixes '*' --source-port-ranges 80 --destination-address-prefixes '*' --destination-port-ranges 3389 --access Allow --protocol Tcp 
```
- assign the NSG to the NIC (Just go the portal then the VM, Networking then Assign NSG too complicated to do CLI)

Configure the Azure Devops agent on the VM following this guide: https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops
```
        [string] $AdminUsername = 'sitecorebuildadmin'

        [string] $AdminPassword = 'PRIVATE'

        Personal Accesss Token: PRIVATE
        RDP IP: 
```
> Do not use the Default Agent Pool, instead create a new Agent Pool (e.g. Docker), as it might give you trouble assigning builds to it: https://developercommunity.visualstudio.com/content/problem/338828/cannot-start-jobs-on-default-agent-pool-could-not.html

> Install the agent as service under `NT AUTHORITY\SYSTEM` otherwise Docker will not run: 
https://github.com/Microsoft/azure-pipelines-tasks/issues/4449

## Use the agent
Finally, configure the correct job pool name (the one containing your self-hosted agent) in `azure-pipelines.yml`.
