<h1 valign="center"><img src="./Assets/cosmosdb.svg"/> Cosmos DB Provider for ASP.NET Core Identity</h1>

[![.NET 6 Build-Test](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/dotnet.yml/badge.svg)](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/dotnet.yml) [![CodeQL](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/codeql-analysis.yml/badge.svg)](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/codeql-analysis.yml)
[![Unit Tests](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/unittests.yml/badge.svg)](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/actions/workflows/unittests.yml)

This is a **Cosmos DB** implementation of an Identity provider for .NET 6 that uses the ["EF Core Azure Cosmos DB Provider."](https://docs.microsoft.com/en-us/ef/core/providers/cosmos/?tabs=dotnet-core-cli) that was forked from [Piero De Tomi's](https://github.com/pierodetomi) excellent project: [efcore-identity-cosmos](https://github.com/pierodetomi/efcore-identity-cosmos).

## Help Find Bugs!

Find a bug? Let us know by contacting us [via NuGet](https://www.nuget.org/packages/AspNetCore.Identity.CosmosDb/2.0.10/ContactOwners) or submit a bug report on our [GitHub issues section](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/issues). Thank you in advance!

# Installation

Add the following [NuGet package](https://www.nuget.org/packages/AspNetCore.Identity.CosmosDb) to your project:

```shell
PM> Install-Package AspNetCore.Identity.CosmosDb
```

Create an [Azure Cosmos DB account](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/create-cosmosdb-resources-portal) - either the serverless or dedicated instance. For testing and development purposes it is recommended to use a 'serverless instance'
as it is [very economical to use](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb#choice-of-cosmos-db-account-type). See [documentation](https://docs.microsoft.com/en-us/azure/cosmos-db/throughput-serverless) to help choose which is best for you.

Set your configuration settings with the connection string and database name. Below is an example of a `secrets.json` file:

```json
{
  "SetupCosmosDb": "true", // Importat: Remove this after first run.
  "CosmosIdentityDbName": "YourDabatabaseName",
  "ConnectionStrings": {
    "ApplicationDbContextConnection": "THE CONNECTION STRING TO YOUR COSMOS ACCOUNT"
  }
}
```

## Update Database Context

Modify the database context to inherit from the `CosmosIdentityDbContext` like this:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace AspNetCore.Identity.CosmosDb.Example.Data
{
    public class ApplicationDbContext : CosmosIdentityDbContext<IdentityUser, IdentityRole>
    {
        public ApplicationDbContext(DbContextOptions dbContextOptions)
          : base(dbContextOptions) { }
    }
}
```

## Modify Program.cs or Startup.cs File

After the "secrets" have been set, the next task is to modify your project's startup file.  For Asp.net
6 and higher that might be the `Project.cs` file. For other projects it might be your `Startup.cs.`

You will likely need to add these usings:

```csharp
using AspNetCore.Identity.CosmosDb;
using AspNetCore.Identity.CosmosDb.Containers;
using AspNetCore.Identity.CosmosDb.Extensions;
using AspNetCore.Identity.Services.SendGrid;
using AspNetCore.Identity.Services.SendGrid.Extensions;
```

Next, the configuration variables need to be retrieved. Add the following to your startup file:

```csharp
// The Cosmos connection string
var connectionString = builder.Configuration.GetConnectionString("ApplicationDbContextConnection");

// Name of the Cosmos database to use
var cosmosIdentityDbName = builder.Configuration.GetValue<string>("CosmosIdentityDbName");

// If this is set, the Cosmos identity provider will:
// 1. Create the database if it does not already exist.
// 2. Create the required containers if they do not already exist.
// IMPORTANT: Remove this setting if after first run. It will improve startup performance.
var setupCosmosDb = builder.Configuration.GetValue<string>("SetupCosmosDb");

```

Add this code if you want the provider to create the database and required containers:

```csharp
// If the following is set, it will create the Cosmos database and
//  required containers.
if (bool.TryParse(setupCosmosDb, out var setup) && setup)
{
    var builder1 = new DbContextOptionsBuilder<ApplicationDbContext>();
    builder1.UseCosmos(connectionString, cosmosIdentityDbName);

    using (var dbContext = new ApplicationDbContext(builder1.Options))
    {
        dbContext.Database.EnsureCreated();
    }
}

```

Now add the database context in your startup file like this:

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseCosmos(connectionString: connectionString, databaseName: cosmosIdentityDbName));
```

Follow that up with the identity provider. Here is an example:

```csharp
builder.Services.AddCosmosIdentity<ApplicationDbContext, IdentityUser, IdentityRole>(
      options => options.SignIn.RequireConfirmedAccount = true // Always a good idea :)
    )
    .AddDefaultUI() // Use this if Identity Scaffolding is in use
    .AddDefaultTokenProviders();
```

## Configure Email Provider (Optional)

When users register accounts or need to reset passwords, you will need (at a minimum), the ability
to send tokens via an [Email provider](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/accconfirm?view=aspnetcore-6.0&tabs=visual-studio#configure-an-email-provider). The example below uses a SendGrid provider. Here is how to add it:

Start by adding the following NuGet package to your project:

```shell
PM> Install-Package AspNetCore.Identity.Services.SendGrid
```

Note: You can sign up for a [free SendGrid account](https://sendgrid.com/) if you do not already have one.

### Configure app to support email

Next we need to configure the application to support our Email provider. Start by adding the following code to your startup file:

```csharp
var sendGridApiKey = builder.Configuration.GetValue<string>("SendGridApiKey");
// Modify 'from' email address to your own.
var sendGridOptions = new SendGridEmailProviderOptions(sendGridApiKey, "foo@mycompany.com");

builder.Services.AddSendGridEmailProvider(sendGridOptions);
```

### Modify "Scaffolded" Identity UI

The example web project uses the "scaffolded" Identity UI. By default it does not use an IEmailProvider.
But in our case we have installed one so we need to modify the UI to enable it.

In your project, find this file:

`/Areas/Identity/Pages/Account/RegisterConfirmation.cshtml.cs`

Find the `OnGetAsync()` method, then look for the following line:

```csharp
DisplayConfirmAccountLink = true;
```

Change that line to `false` like the following:

```csharp
DisplayConfirmAccountLink = false;
```

# Complete Startup File Example

The above instructions showed how to modify the startup file to make use of this provider. Sometimes
it is easier to see the end result rather than peicemeal.  Here is an example Asp.Net 6 Project.cs
file configured to work with this provider, scaffolded identity web pages, and the SendGrid email provider:

- [Program.cs](/AspNetCore.Identity.CosmosDb.Example/Program.cs)

An [example website](https://github.com/CosmosSoftware/AspNetCore.Identity.CosmosDb/tree/master/AspNetCore.Identity.CosmosDb.Example) is available for you to download and try.

# Supported LINQ Operators User and Role Stores

Both the user and role stores now support queries via LINQ using Entity Framework.
Here is an example:

```csharp
var userResults = userManager.Users.Where(u => u.Email.StartsWith("bob"));
var roleResults = roleManager.Roles.Where (r => r.Name.Contains("water"));
```

For a list of supported LINQ operations, please see the ["Supported LINQ Operations"](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/sql-query-linq-to-sql#SupportedLinqOperators)
documentation for more details.

# Changelog

## v1.0.6

- Introduced support for `IUserLoginStore<TUser>` in User Store

## v1.0.5

- Introduced support for `IUserPhoneNumberStore<TUser>` in User Store

## v1.0.4

- Introduced support for `IUserEmailStore<TUser>` in User Store

## v2.0.0-alpha

- Forked from source repository [pierodetomi/efcore-identity-cosmos](https://github.com/pierodetomi/efcore-identity-cosmos).
- Refactored for .Net 6 LTS.
- Added `UserStore`, `RoleStore`, `UserManager` and `RoleManager` unit tests.
- Namespace changed to one more generic: `AspNetCore.Identity.CosmosDb`
- Implemented `IUserLockoutStore` interface for `UserStore`

## v2.0.1.0

- Added example web project

## v2.0.5.1

- Implemented IQueryableUserStore and IQueryableRoleStore

# Unit Test Instructions

To run the unit tests you will need two things: (1) A Cosmos DB Account, and (2) a connection string to that account.
Here is an example of a `secrets.json` file created for the unit test project:

```json
{
  "ConnectionStrings": {
    "ApplicationDbContextConnection": "AccountEndpoint=YOURCONNECTIONSTRING;"
  }
}
```

## Choice of Cosmos DB Account Type

Because this implementation uses multiple containers, it exceeds the minimum throughput requirements for the "free" Cosmos DB
account.

However, if you deploy a "[serverless](https://docs.microsoft.com/en-us/azure/cosmos-db/throughput-serverless)" account, your
monthly costs will be very low. For example, our serverless Cosmos account used for unit testing costs under $0.10 (US) a month as
of July 2022.

# References

To learn more about Asp.Net Identity and items realted to this project, please see the following:

- [.Net 5 version of AspNetCore.Identity.Cosmos (pierodetomi/efcore-identity-cosmos)](https://github.com/pierodetomi/efcore-identity-cosmos)
- [Asp.Net Core Identity on GitHub](https://github.com/dotnet/AspNetCore/tree/main/src/Identity)
- [Introduction to Identity on ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-6.0&tabs=visual-studio)
  - [Account confirmation and password recovery in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/accconfirm?view=aspnetcore-6.0&tabs=visual-studio)
- [Supported LINQ Operaions for Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/sql/sql-query-linq-to-sql#SupportedLinqOperators)
