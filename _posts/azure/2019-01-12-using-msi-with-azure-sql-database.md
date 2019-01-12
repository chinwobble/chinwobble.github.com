---
layout: post
tags: [azure, sql-server, c#, dotnet]
---
{% include JB/setup %}
This post describes how to use Managed Service Identity to connect a .NET Application hosted as an _Azure App Service_ to an _Azure SQL Database_.

Managed Service Identity (MSI) is a Microsoft Azure feature that allows Azure resources to authenticate / authorise themselves with (_other supported azure resources_)[https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/services-support-msi]. The appeal is that secrets such as database passwords are not required to be copied onto developers' machines or checked into source control.

## Instructions 

### Step 1: Install the required Packages
You will need to install the below packages to correctly identify your app to the Azure SQL Database.
```
Install-Package Microsoft.Azure.Services.AppAuthentication
```
This package will allow your application to obtain an access token from from _Azure Active Directory_. 
- When your app is running from a developer's machine, it will use the developoer's Microsoft Identity obtained from Visual Studio or the (Azure CLI)[https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest]. 
- When the application is running on your app service, it will use the managed identity assigned to it.

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

If SqlConnection doesn't have the `AccessToken` property, you can either install the lastest version of `System.Data.SqlClient` or target a newer version .NET.

If your app uses Entity Framework 6, you can use a less well known feature - (interceptors)[https://docs.microsoft.com/en-us/ef/ef6/fundamentals/logging-and-interception] to add the Access Token every time Entity Framework attempts to open a connection to the dataase. 
**Note** that this feature is still not available in EF Core 2.2.  
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
    ...
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

### Step 3: Updte your connection strings
You can now remove username and password from your connection strings.
```
connectionString="Server=tcp:database-server.database.windows.net,1433;Database=my_database"
```

### Step 4: Assign a Managed Identity to the Azure App Service
You will need to configure the app service to assign a managed identity. This can be done using the 
[Azure Portal, ARM templates, Powershell or using the Azure CLI](https://docs.microsoft.com/en-us/azure/app-service/overview-managed-identity).

If your app uses deployment slots they will also need managed identities individually assigned to them.

### Step 5: Create the database user with the correct permissions
To connect to Azure SQL Databases you will only need to create a _contained user_. This differs from on-premises SQL Server instances that require both a server login and a database user. The below SQL shows how to create a contained user with some roles to read and write from the database.
```
create user [my-app-service] from external provider;
alter role db_datareader add member [my-app-service];
alter role db_datawriter add member [my-app-service];
create user [my-app-service/slots/staging] from external provider;
alter role db_datawriter add member [my-app-service/slots/staging];
alter role db_datawriter add member [my-app-service/slots/staging];
```
**Note** that you will need to configure a Server Admin for the Azure SQL Server resource. 

You will also need to the correct roles to create the contained users - identity the relevant role is left as a task for readers. 

## References
[Adding Users to Azure SQL Databases](https://www.mssqltips.com/sqlservertip/5242/adding-users-to-azure-sql-databases/)

[What is managed identities for Azure resources?](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)

[Tutorial: Secure Azure SQL Database connection from App Service using a managed identity](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-connect-msi)