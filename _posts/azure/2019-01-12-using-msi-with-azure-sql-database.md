---
layout: post
tags: [azure, sql-server, c#, dotnet]
---
{% include JB/setup %}
Managed Service Identity (MSI) is Microsoft Azure's preview for for that allows azure resources to authenticate / authorise themselves with _other supported azure resources_. The appeal is that secrets such as database passwords are not required to copied onto developers' machines or checked into source control.

This post describes how to use MSI to connect an ASP.NET web application hosted as an Azure App Service to an *Azure SQL Database*.

## Instructions 

### Step 1: Install the required Packages
YOu will need to install the below packages to correctly identify your app to SQL Server.
```
Install-Package Microsoft.Azure.Services.AppAuthenticationthe 
```
This first package will obtain the token from the azure environment. 

### Step 2: Update the code to use the packages 
If your application is already directly using `System.Data.SqlClient.SqlConnection` then you simply need to change your connection factory to set the AccessToken property _before_ it opens the database connection.
If one of your environments is connecting to a database that is not hosted as an Azure SQL Database (e.g. localDB for local development) you may also want to add an if statement check around setting the AccessToken as you cannot mix MSI with other forms of authentication.

```
    // this token provivder should be registered as a singleton 
    var tokenProvider = AzureServiceTokenProvider();
    ...
    var dbConnection = new SqlConnection(connectionString);
    if (IsAzureDB)
    {
        db.AccessToken = await tokenProvider.GetAccessTokenAsync("https://database.windows.net/");
    }
    return dbConnection;
```

If your application is using Entity Framework 6, you can register a database connection intercetpor that will automatically add the Access Token every time Entity Framework attempts to open a connection to the dataase. 
*Note* that this feature is still not available in EF Core 2.2.  
```
[DebuggerStepThrough]
public class AzureDBConnectionInterceptor : IDbConnectionInterceptor
{
    public void Opening(DbConnection connection, DbConnectionInterceptionContext interceptionContext)
    {
        var sqlConnection = connection as SqlConnection;
        sqlConnection.AccessToken = tokenProvider.GetAccessTokenAsync("https://database.windows.net/")GetAwaiter().GetResult();
    }
    // other interface methods removed for brevity 
}

public class MyDbConfiguration : DbConfiguration
{
    public MyDbConfiguration()
    {
        if (ConfigManager.IsAzureDB)
        {
            AddInterceptor(new AzureDBConnectionInterceptor());
        }
    }
}
```
### Step 3: Create the database user with the correct permissions


## References
[What is managed identities for Azure resources?](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)

[Tutorial: Secure Azure SQL Database connection from App Service using a managed identity](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-connect-msi)