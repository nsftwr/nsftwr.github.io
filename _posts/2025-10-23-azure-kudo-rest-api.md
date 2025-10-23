---
title: Send commands to Azure App Service hosts using REST API
description: Interact with Azure App Service via REST API like you would usually interact through Kudu.
date: 2025-10-23 23:00:00 +0000
categories: [Azure, Automation]
tags: [azure, automation]
pin: true
image:
  path: /assets/img/posts/azure-kudo-rest-api/thumbnail.png
---

## Introduction

Have you ever needed to access multiple App Services through the Advanced Tools (kudu) console in Azure? If you are dealing with 10's of apps it can easily eat up a lot of your time, especially if you are dealing with something as simple as I did today - extracting the precise runtime versions for nearly 100 App Services.

## Interacting

Interacting is as easy as any other REST API - it is a POST call to the SCM endpoint of your app. It expects a JSON payload, where you can pass in the `cmd` console command.

```powershell
Invoke-RestMethod 'https://<app-name>.scm.azurewebsites.net/api/command' `
  -Method 'POST' `
  -Headers $headers `
  -Body '{"command": "dotnet --version"}'
```

Additionally, you can also pass in the directory where you want the command to take place with the `dir` property, as such:

```powershell
Invoke-RestMethod 'https://<app-name>.scm.azurewebsites.net/api/command' `
  -Method 'POST' `
  -Headers $headers `
  -Body '{"command": "ls", "dir":"site\\wwwroot"}'
```

### Authentication

- JWT Token (preferred)

A bearer token can be used to authenticate to the API. Keep in mind that you will need to have sufficient RBAC roles assigned, such as `Website Contributor`.

```powershell
$token = az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv
$headers = @{
    "Authorization" = "Bearer $token"
}
```

- Basic Authentication

Whilst basic authentication should be disabled as per [Azure Well-Architected Framework best practices](https://github.com/MicrosoftDocs/well-architected/blob/7c88fc17eccd1078f20111cb42d4acad571ae48b/well-architected/service-guides/app-service-web-apps.md), it is an option that can be used to interact with the App Service.

> We don't recommend basic authentication as a secure deployment method. Microsoft Entra ID employs OAuth 2.0 token-based authentication, which provides numerous advantages and enhancements that address the limitations that are associated with basic authentication.

```powershell
$password = az webapp deployment list-publishing-credentials --name <app-name> --resource-group <resource-group-name> --query 'publishingPassword' --output tsv
$username = "<app-name>" # Pay attention to the $ at the start - that's important

$creds = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("`$$($username):$($password)")) # Pay attention to the $ in front of the username - that's important
$headers = @{
    "Authorization" = "Basic $token"
}
```

## Code Snippet

Whilst the example I gave above was my challange today, I can remember many times it would have been useful to interact with App Services via a script through the API, rather manually accessing it via kudu or tunneling into it.

One of the scripts I use frequently across multiple App Services (and multiple environments) is the [Test-SqlConnection.ps1](https://github.com/nsftwr/toolbox/blob/main/scripts/App-Sql-Connection-Tester/Test-SqlConnection.ps1) script. It's a little script I created in order to test the following:

- The App Service can resolve SQL FQDN;
- The App Service can request a Bearer token as its Managed Identity;
- The App Service can connect and query the database.

It allows me to troubleshoot and verify connectivity between the App Service and SQL Server before I hand it over to the developers.

```powershell
$environment = @("dev", "tst", "prd")
$token = az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv

$headers = @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer $token"
}

foreach ($env in $environment) {
    $app = "app-$env-client-01"
    $sql = "sql-$env-workload-01"
    $db  = "sqldb-$env-workload-01"

    $url = "https://$app.scm.azurewebsites.net/api/command"

    Write-Host "Querying $app..."
    
    $body = @{
        command = @(
            "curl -O https://raw.githubusercontent.com/nsftwr/toolbox/main/scripts/App-Sql-Connection-Tester/Test-SqlConnection.ps1",
            "powershell ./Test-SqlConnection.ps1 -Server $sql -Database $db",
            "rm ./Test-SqlConnection.ps1"
        ) -join " && "
    } | ConvertTo-Json
    
    (Invoke-RestMethod $url -Method 'POST' -Headers $headers -Body $body).Output
}
```

And at the end, it should have the following outputs:

- Success
![Successful connection](/assets/img/posts/azure-kudo-rest-api/successful_run.png)

- Failure
![Failed connection](/assets/img/posts/azure-kudo-rest-api/failed_run.png)