# Thycotic SDK Integration Doc

- [SDK Integration in C# project](#sdk-integration-in-C#-project)
  - [Prerequisites](#prerquisites)
  - [Configuring the SDK](#configuring-the-sdk)
  - [Usage](#usage)
- [SDK Integration in web.config](#sdk-integration-in-webconfig)
  - [Prerequisites - Web](#prerequisites---web)
  - [Configuring the SDK - Web](#configuring-the-sdk---web)
  - [Usage - Web](#usage---web)

## SDK Integration in C# project

In this scenario we’re assuming we can recompile the application, and will dynamically retrieve passwords from the vault whenever needed

### Prerequisites

1. .NET Standard 2.0 which equals to
  - .NET Core 2.0 if building a .NET Core application
  - .NET (Full Framework) 4.6.1
1. Create a rule in Secret Server for client onboarding:
   1. Navigate to Admin > SDK Client Management > Client Onboarding Tab
   1. Click on the “+ Rule” button to create a new Rule
   1. Name your Rule (Something that helps identify what this is, e.g. the application name)
   1. Assign IPv4 restrictions (optional)
   1. Assign an Application User Account (create one if you haven't already)
   1. Generate a Rule Key (optional)
   1. Save
1. The application account you created needs to have access to the Secrets your application needs. Ensure the permissions are accurate at the folder/secret level
1. Download the SDK files from NuGet. You can do this either from NuGet.org or directly from Visual Studio
   1. Search for Thycotic
   2. Install the following packages:
    - Thycotic.SecretServer.SDK
    - Thycotic.SecretServer.SDK.Extension.Configuration
    - Thycotic.SecretServer.SDK.Extension.Integration
1. In your Project add the following references:

```C#

  using Thycotic.SecretServer.Sdk.Extensions.Integration.Clients;
  using Thycotic.SecretServer.Sdk.Extensions.Integration.Models;
  using Thycotic.SecretServer.Sdk.Infrastructure.Models;
```

Alternatively, you can instantiate the objects and let Visual Studio add the references for you.

### Configuring the SDK

You need to instantiate a new SecretServerClient object to interact with the SDK. Below is a sample code:

``` C#
var client = new SecretServerClient();
//configure if not configured
client.Configure(new ConfigSettings
  {
    SecretServerUrl = string,
    RuleName = string,
    RuleKey = string,
    CacheStrategy= CacheStrategy.enum,
    CacheAge= int,
    ResetToken= string
  });
  ```
The code above will register the integrated client with Secret Server. Below is an explanation of the properties of the client's ConfigSettings:
- `ConfigSettings` is an object
- `SecretServerUrl` is the base URL for your Secret Server (if you have a Load Balancer then it should be the load balanced URL
- `RuleName` is the name of the rule we created earlier
- `RuleKey` is the Key we generated, can be NULL if no key
- `CacheStrategy` is how the SDK will cache requests. This is an enum with four options:
  - `CacheThenServer`
  - `CacheThenServerAllowExpired` (This will allow fallback; in case Secret Server isn’t available and the cache is expired, the SDK will still use the cache until it can contact Secret Server and prime the cache)
  - `Never`
  - `ServerThenCache` (This mode is for redundancy since it will fall back to cache in case Secret Server isn’t available)
- `CacheAge` is how long the cache is valid before it expires in minutes
- `ResetToken`  is a random string to revoke clients and reinitialize them

### Usage

Now we can use the client to get a Secret, Token, or a Secret field from Secret Server, and feed that to our application
```c#

var password = client.GetSecretField(<SECRET ID>,"password"); //replace <SECRET ID>

```
The code above will retrieve a password from Secret Server, which we can then pass to a connection string or anywhere a password is needed.

You can call the ```GetSecret();``` method on the client object to get the full Secret object, and then access the items property which holds a collection of the Secret fields and their values. How you access these values is up to you, but you can use Linq to query what you’re looking for, e.g.
```C#

var secret = client.GetSecret(<SECRET ID>); //replace <SECRET ID>
var server = secret.Items.First(x => x.Slug == "server").ItemValue;
var username = secret.Items.First(x => x.Slug == "username").ItemValue;
var password = secret.Items.First(x => x.Slug == "password").ItemValue;
var database = secret.Items.First(x => x.Slug == "database").ItemValue;

```
and then use these variables to build a connection string:
```C#

SqlConnectionStringBuilder builder = new SqlConnectionStringBuilder(GetConnectionString())
{
    ConnectionString = $"server={server};user id={username};password={password};initial catalog={database}"
};

```

Here is the complete code sample:
```C#

using System;
using System.Linq;
using Thycotic.SecretServer.Sdk.Extensions.Integration.Clients;
using Thycotic.SecretServer.Sdk.Extensions.Integration.Models;
using Thycotic.SecretServer.Sdk.Infrastructure.Models;
using System.Data.SqlClient;

namespace SDK.Integration
{
    class Program
    {
        static void Main(string[] args)
        {
            var url = ""
            var ruleName = ""
            var ruleKey = ""
            var cacheAge = int //replace with number for cache age
            var secretId = int //replace with Secret ID
            var client = new SecretServerClient();
            //configure if not configured
            client.Configure(new ConfigSettings
            {
                SecretServerUrl = url,
                RuleName = ruleName,
                RuleKey = ruleKey,
                CacheStrategy= CacheStrategy.CacheThenServerAllowExpired,
                CacheAge= cacheAge, //in minutes
                ResetToken= "Token"
            });
            var secret = client.GetSecret(secretId);
            var server = secret.Items.First(x => x.Slug == "server").ItemValue;
            var username = secret.Items.First(x => x.Slug == "username").ItemValue;
            var password = secret.Items.First(x => x.Slug == "password").ItemValue;
            var database = secret.Items.First(x => x.Slug == "database").ItemValue;

            //connection string example
            SqlConnectionStringBuilder builder = new SqlConnectionStringBuilder(GetConnectionString())
            {
                ConnectionString = $"server={server};user id={username};password={password};initial catalog={database}"
            };

            Console.WriteLine(builder.ConnectionString);
            Console.ReadKey();
        }
        private static string GetConnectionString()
        {
            return "Server=(local);Integrated Security=SSPI;" +
                "Initial Catalog=AdventureWorks";
        }
    }
}

```

## SDK Integration in web.config

In this scenario we’re assuming we can't recompile the application, or prefer not to. The use case is as follows:
- ASP.NET web application and is a .NET Standard 2.0 application
  - We have a ConnectionString(s) inside of our config file that contains plaintext passwords
  - We have appSettings with plaintext passwords that our app uses to connect to external services
The SDK will allow us to pull data from Secret Server and inject it in the config file.

### Prerequisites - Web

1. Create a rule in Secret Server for client onboarding:
   1. Navigate to Admin > SDK Client Management > Client Onboarding Tab
   1. Click on the “+ Rule” button to create a new Rule
      1. Name your Rule
      1. Assign IPv4 restrictions (optional)
      1. Assign an Application User Account (create one if you haven't already)
      1. Generate a Rule Key (optional)
      1. Save
1. Download the dlls from [Nuget](https://www.nuget.org/packages?q=thycotic "https://www.nuget.org/packages?q=thycotic")
1. After downloading, use 7zip to unpack the ngpkg and navigate to the lib directory
1. Make sure you only extract dlls from subdirectories in NetStandard2.0 or net461
1. Copy the extracted dlls to your application’s bin folder:
    - Thycotic.SecretServer.Sdk.dll
    - Thycotic.SecretServer.Sdk.Extensions.Configuration.dll
    - Thycotic.SecretServer.Sdk.Extensions.Integration.dll
    - Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule.dll

### Configuring the SDK - Web

- Open you web.config file in your preferred code editor and add the inside your appSettings tag following:
  ```xml

  <appSettings>
      <add key="tss:CacheAge" value="<cache-age>" />
      <add key="tss:CacheStrategy" value="<cache-strategy>" />
      <add key="tss:SecretServerUrl" value="<your-secret-server-url>" />
      <add key="tss:RuleName" value="<rule-name>" />
      <add key="tss:RuleKey" value="<rule-key>" />
      <add key="tss:ResetToken" value="<reset-token>" />
  </appSettings>

  ```
  - This will configure the SDK to talk to Secret Server, attach it to a rule, authenticate with the optional pre-shared key, configure caching, and add a reset token for reinitialization. Below is an explanation of these key-value pairs:
    - tss:CacheAge: cache age in minutes. i.e. how long should the SDK keep the cache before trying to refresh
    - tss:CacheStrategy: should the SDK cache or not? Strategies are numbered 0 – 3
      - 0: Never cache
      - 1: Server then cache
      - 2: Cache then server
      - 3: Cache then server, but allow expired cache if the server is unreachable
    - SecretServerUrl: Your Secret Server Url
    - tss:RuleName: the name of the rule you created in Secret Server
    - tss:RuleKey:  the pre-shared key you generated for the rule
    - tss:ResetToken: This is a string value used to reinitialize the SDK. It can be anything, and changing it will cause the client to reinitialize and reregister itself
- Scroll to
    ```xml
          <system.webServer>
                <module>
    ```
  tag and add the following:
    ```xml
    <remove name="ThycoticInterceptModule" />
    <add name="ThycoticInterceptModule"
    type="Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule.Modules.ThycoticInterceptModule,Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule" />
    ```
- Our section should look like this:
  ```xml

  <system.webServer>
      <validation validateIntegratedModeConfiguration="false" />
      <modules>
        <remove name="ThycoticInterceptModule" />
        <add name="ThycoticInterceptModule"
            type="Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule.Modules.ThycoticInterceptModule,Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule" />
        <remove name="TelemetryCorrelationHttpModule" />
        <add name="TelemetryCorrelationHttpModule"
            type="Microsoft.AspNet.TelemetryCorrelation.TelemetryCorrelationHttpModule, Microsoft.AspNet.TelemetryCorrelation"
            preCondition="integratedMode,managedHandler" />
        <remove name="ApplicationInsightsWebTracking" />
        <add name="ApplicationInsightsWebTracking"
            type="Microsoft.ApplicationInsights.Web.ApplicationInsightsHttpModule, Microsoft.AI.Web"
            preCondition="managedHandler" />
      </modules>
    </system.webServer>
    ```
> Important: You may need to add a binding redirect if you run into:
>
>```Message: System.IO.FileLoadException : Could not load file or assembly 'System.Runtime.InteropServices.RuntimeInformation, Version=0.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a' or one of its dependencies. The located assembly's manifest definition does not match the assembly reference. (Exception from HRESULT: 0x80131040)```
>
> Simply add the following to your web.config file
>

```xml

<dependentAssembly>
<assemblyIdentity name="System.Runtime.InteropServices.RuntimeInformation"    publicKeyToken="b03f5f7f11d50a3a" culture="neutral" />
<bindingRedirect oldVersion="0.0.0.0-4.0.2.0" newVersion="4.0.2.0" />
</dependentAssembly>

```
> Match the version number with what you have installed on your system

## Usage - Web

To replace plaintext credentials in your web.config file we simply use string interpolation to replace the hard-coded values with Secret Server values as shown below

Example old ConnectionString:
```xml

<connectionStrings>
    <clear />
    <add
        name="AdventureWorks2014ConnectionString"
        connectionString="Data Source=sql01.domain.com;Initial Catalog=AdventureWorks2014;Persist Security Info=True;User ID=sa;Password=SgW#5)zLpo($@d"
        providerName="System.Data.SqlClient"
    />
  </connectionStrings>
  ```

New connection string with Secret Server SDK"

```xml

<connectionStrings>
    <clear />
    <add 
        name="AdventureWorks2014ConnectionString"
        connectionString="Data Source=${server};Initial Catalog=${database};Persist Security Info=True;User ID=${username};Password=${password}?3112"
        providerName="System.Data.SqlClient"
    />
  </connectionStrings>
```

Where the `3112` is the Secret Id preceded by `?`
