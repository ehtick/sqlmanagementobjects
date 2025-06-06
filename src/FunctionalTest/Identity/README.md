# Microsoft.SqlServer.ADO.Identity package

This package provides implementations of two commonly needed authentication providers to support OpenId federated credentials in Azure Devops.

## TokenCredential

`Microsoft.SqlServer.ADO.Identity.AzureDevOpsFederatedTokenCredential`

This `TokenCredential` implementation wraps the ADO  `oidctoken` API to generate tokens from service connections that have federated enterprise application credentials in Azure. Use this class in your test or pipeline utility code when tokens generated by the `AzureCLI@2`task are too short lived.

## SqlAuthenticationProvider

To access Azure SQL database and Azure SQL Managed Instance resources using your enterprise app credential or managed identity in an ADO agent, use the `AzureDevOpsSqlAuthenticationProvider` class. This provider can be registered to handle `SqlAuthenticationMethod.ActiveDirectoryServicePrincipal`, `SqlAuthenticationMethod.ManagedIdentity`, and `SqlAuthenticationMethod.ActiveDirectoryDefault` authentication methods, following the directions at https://learn.microsoft.com/sql/connect/ado-net/sql/azure-active-directory-authentication?view=sql-server-ver16#support-for-a-custom-sql-authentication-provider

### Create a federated credential for your service connection

For ADO service connections using an enterprise application, go to the `Certificates & secrets` tab in Azure portal.
Selected `Federated credentials`.
Add a credential with these settings:

Issuer: `https://vstoken.dev.azure.com/<ADO project id>`
Subject identifier: `sc://<ADO org name>/<ADO project name>/<service connection name>`

Example, for the `DacFxTests` service connection in the msdata SQLToolsAndLibraries project:

Issuer: `https://vstoken.dev.azure.com/8b119ea1-2e2a-4839-8db7-8c9e8d50f6fa`
Subject identifier: `sc://msdata/SQLToolsAndLibraries/DacFxTests`

### Add the service connection to your pipeline

Use a standard task like `AzureCLI@2` to tell ADO to expose the service connection endpoint to your pipeline. Skip this step if you are using managed identity.

Example template:

```yml
# Sets environment variables for client id and tenant id based on the given service connection name
parameters:
  - name: smoTestServiceConnectionName
    type: string
    default: 'DacFxTests'
  - name: clientIdVarName
    type: string
    default: 'AZURE_CLIENT_ID'
  - name: tenantIdVarName
    type: string
    default: 'AZURE_TENANT_ID'

steps:
  - task: AzureCLI@2
    inputs:
      addSpnToEnvironment: true
      azureSubscription: ${{ parameters.smoTestServiceConnectionName }}
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        Write-Host "##vso[task.setvariable variable=${{ parameters.clientIdVarName }};]$env:servicePrincipalId"
        Write-Host "##vso[task.setvariable variable=${{ parameters.tenantIdVarName }};]$env:tenantId"

```

### Add the TokenCredential or SqlAuthenticationProvider implementation to your C# code

This example snippet sets up an AKV client whose token provider chain includes the AzureDevOpsFederatedTokenCredential as the last option.

```C#
// prefer local user on dev machine over the certificate
var credentials = new List<Azure.Core.TokenCredential>() { new DefaultAzureCredential()};
foreach (var thumbprint in CertificateThumbprints ?? Enumerable.Empty<string>())
{
    var certificate = FindCertificate(thumbprint);
    if (certificate != null)
    {
        credentials.Add(new ClientCertificateCredential(AzureTenantId, AzureApplicationId, certificate));
    }
    break;
}
credentials.Add(new AzureDevOpsFederatedTokenCredential(new AzureDevOpsFederatedTokenCredentialOptions() { TenantId = AzureTenantId, ClientId = AzureApplicationId }));
var credential = new ChainedTokenCredential(credentials.ToArray());
secretClient = new SecretClient(new Uri($"https://{KeyVaultName}.vault.azure.net"), credential);
```

### Configuration

The `AzureDevOpsFederatedTokenCredentialOptions` class provides configuration to the `TokenCredential` implementation. It uses environment variables for its default configuration, which should satisfy most use cases. The variables your pipeline should set explicitly are these:
 - `AZURE_CLIENT_ID` : the app id guid
 - `AZURE_TENANT_ID` : the tenant id guid of the app
 - `SERVICE_CONNECTION_ID` : the resource identifier guid of the service connection. Can be found in the ADO portal from the URL of the service connection, or extracted from an `ENDPOINT_DATA_<guid>_<varname>` environment variable in the `AzureCLI@2` pipeline task that references your service connection.
 - `SYSTEM_ACCESSTOKEN` : usually provided by passing `$(System.AccessToken)` from the pipeline

 To use a user assigned managed identity instead of a federated service principal, set the variable `AZURE_IDENTITY_CLIENT_ID` to the client id of a user-assigned managed identity assigned to your test agent.