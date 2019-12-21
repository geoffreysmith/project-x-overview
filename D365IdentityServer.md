# Introduction 
Sitecore D365 Identity Server with documentation for installation on D365 server. Refer to: D365 - SITESUPP-Setupnewenvironment-270919-1924-3950.pdf

# Getting Started
Will need to be installed on each D365 environment once.

# Basic Operation

1. App Settings contain:
	Client ID: EnvironmentName (can be arbitrary but easier to refer to it as the whatever environment is)
	Client Secret: Password (can be arbitrary but I've just been using alphanumberic with no special characters)
	Redirect URLs: There are two, each corresponds to the Sitecore URL of whatever environment should be calling it.

2. Reasoning
 - Non-Sitecore apps can call Client ID and Secret to get information. For example, this is how the iniital catalog is populated in a console application (not related to this).
 - Sitecore itself uses it to authenticate for pricing and order information
 - Customers login through Sitecore, directed to the server, then redirected back to Sitecore, the reason for the redirect URLs.
 - Customers are in Sitecore as "virtual users" but actually stored in Dynamics for operations like carts and orders, among other things.

 # Basic Setup

Repo:
https://SGIUSA@dev.azure.com/SGIUSA/Dynamics365/_git/D365SitecoreIdentityServer

## Server Install

Step 1 Install Server:
```Import-Module WebAdministration
$cert =  dir cert: -Recurse | Where-Object { $_.Subject -like "*.cloud.dynamics.com" }
$Hostname 'dynamics365devaos.cloudax.EXAMPLE.com' //this is example depending on host
New-WebBinding -Name IdentityServer -IP * -Port 5000 -Protocol https -HostHeader $HostName -SslFlags 1 -PhysicalPath K:\IdentityServer
New-Item -Path “IIS:\SslBindings\!5000!IdentityServer -Thumbprint $cert.Thumbprint -SSLFlags 1
Set-ItemProperty IIS:\AppPools\IdentityServer -name IdentityServer -value @{userName="domainifexists\localadminaccount";password="password";identitytype=3}
New-NetFirewallRule -DisplayName "Allow Identity Server" -Direction Inbound -LocalPort 5000 -Protocol TCP 
```
Step 2: Place the compiled site into where the server was installed

Step 3: Configure Azure nsg
```
az network nsg rule create -g sgiusaqa --nsg-name sgiusaqa-networksecuritygroup -n IdentityServer --source-address-prefixes Internet --priority 300 --destination-port-ranges 443 5000 --access Allow --protocol Tcp --description "Allow Internet to Identity ServerASG on ports 80,8080."
```
Step 4: Configure Load Balancer
```
az network lb inbound-nat-rule create -g sgiusaqa -n IdentityServer --lb-name dev1f2923e9f1-loadbalancer --protocol Tcp --frontend-port 5000 --backend-port 5000
```

Step 5: Azure AD App Creation
```
PS C:\Windows\system32> az ad app create --display-name sgiusaqadeSyncTool `
                  --available-to-other-tenants false `
                  --credential-description SitecorePrivate `
                  --end-date 2299-12-31 `
                  --key-type Password `
                  --password @^CY7VgBukbXNh9 `
                  --reply-urls https://sgiusaqac7dc2bff6c7a66addevret.cloudax.dynamics.com/auth `
                  --oauth2-allow-implicit-flow false

```
Step 6:

Patch some mistakes Azure CLI makes, first get the object id of ghe Azure App:
```
az ad app update --id e042ec79-34cd-498f-9d9f-123456781234 --app-roles @manifest.json
("manifest.json" contains the following content)
[{
	"oauth2AllowIdTokenImplicitFlow":false"
}]
```

Remove membership section, disable it first:
```
az ad app update --id e042ec79-34cd-498f-9d9f-123456781234 --app-roles @manifest.json
("manifest.json" contains the following content)
[{
	  "oauth2Permissions":[
    {
      "adminConsentDescription": "Allow access to read all users' todo items.",
      "adminConsentDisplayName": "Read access to todo items",
      "id": "43dc1069-125f-4aac-b554-7a837e049ed1",
      "isEnabled": false,
      "type": "User",
      "userConsentDescription": "Allow access to read your todo items.",
      "userConsentDisplayName": "Read access to your todo items",
      "value": "Todo.Read"
    }
  ]
}]
```
Final clean-up
```
az ad app update --id e042ec79-34cd-498f-9d9f-123456781234 --app-roles @manifest.json
("manifest.json" contains the following content)
[{	  
	"oauth2Permissions":[]
}]
```