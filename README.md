# Thycotic SDK Integration Doc

- [SDK Integration in C# project](#dk-integration-in-c-project)
  - [Prerequisites](#prerequisites)
  - [Configuring the SDK](#configuring-the-sdk)
  - [Usage](#usage)
- [SDK Integration in web.config (NetStandard 2.0)](#sdk-integration-in-webconfig)
  - [Prerequisites - Web](#prerequisites---web)
  - [Configuring the SDK - Web](#configuring-the-sdk---web)
  - [Usage - Web](#usage---web)
- [SDK API](#sdk-api)
- [SDK Client](#sdk-client)

## SDK Integration in C# project

In this scenario we’re assuming we can recompile the application, and will dynamically retrieve passwords from the vault whenever needed.

### Prerequisites

1. Create a rule in Secret Server for client on boarding:
   1. Navigate to Admin > SDK Client Management > Client On boarding Tab
   2. Click on the “+ Rule” button to create a new Rule
   3. Name your Rule (Something that helps identify what this is, e.g. the application name)
   4. Assign IPv4 restrictions (optional)
   5. Assign an Application User Account (create one if you haven't already)
   6. Generate a Rule Key (optional)
   7. Save
1. The application account you created needs to have access to the Secrets your application needs. Ensure the permissions are accurate at the folder/secret level.
1. Download the SDK packages from NuGet. You can do this either directly from Visual Studio (recommended), or manually download and install them from [Nuget.org](https://www.nuget.org/packages?q=thycotic "https://www.nuget.org/packages?q=thycotic"). **(The latest version is always recommended unless explicitly told otherwise)**
   1. Search for Thycotic
   1. Install the following packages:
      - Thycotic.SecretServer.SDK
      - Thycotic.SecretServer.SDK.Extension.Configuration
      - Thycotic.SecretServer.SDK.Extension.Integration
   1. Note: The SDK Nuget packages target both .NET Framework v4.5 or higher, or .NET Standard 2.0. Visual Studio will install the appropriate version based on your project type.
      - .NET Standard 2.0 supports either:
        - .NET Core 2.0 if building a .NET Core application
        - .NET (Full Framework) 4.6.1
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
- `SecretServerUrl` is the base URL for your Secret Server (if you have a Load Balancer then it should be the load balanced URL)
- `RuleName` is the name of the rule we created earlier
- `RuleKey` is the Key we generated, can be NULL if no key
- `CacheStrategy` is how the SDK will cache requests. This is an enum with four options:
  - `CacheThenServer`
  - `CacheThenServerAllowExpired` (This will allow fallback; in case Secret Server isn’t available and the cache is expired, the SDK will still use the cache until it can contact Secret Server and prime the cache)
  - `Never`
  - `ServerThenCache` (This mode is for redundancy since it will fall back to cache in case Secret Server isn’t available)
- `CacheAge` is how long the cache is valid before it expires in minutes
- `ResetToken`  is a random string to revoke clients and reinitialize them

Once the client is configured for the first time, a series of encrypted configuration files are created. By default they will be saved in the current working directory of your application. This path can be customized with the SecretServerSdkConfigDirectory AppSetting. 

If using the .NET Standard version of the SDK, these config files are protected by an encryption key. It is saved in the current user's home directory by default, but this path can also be customized with the SecretServerSdkKeyDirectory AppSetting if necessary.

If you need to change your configuration, change your reset token and the SDK will reinitialize the config files. Otherwise, the SDK will not reconfigure itself as long as it can still access and decrypt the config files. Calls to client.Configure() are ignored if the reset token remains the same and the client has already been configured.

### Usage

Now we can use the client to get a Secret, Token, or a Secret field from Secret Server, and feed that to our application.

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
            var url = "";       //replace with Secret Server URL
            var ruleName = "";  //replace with SDK rule name
            var ruleKey = "";   //replace with on boarding key
            var cacheAge = int; //replace with number for cache age
            var secretId = int; //replace with Secret ID
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

Note that the injected data is not available in Application_Start, because this is a HTTP Intercept module it will not have ran yet.

### Prerequisites - Web

1. Create a rule in Secret Server for client onboarding:
   1. Navigate to Admin > SDK Client Management > Client Onboarding Tab
   1. Click on the “+ Rule” button to create a new Rule
      1. Name your Rule
      1. Assign IPv4 restrictions (optional)
      1. Assign an Application User Account (create one if you haven't already)
      1. Generate a Rule Key (optional)
      1. Save
1. Install the Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule Nuget package.
  - This is available in the Nuget package manager from Thycotic. Installing it will also install the necessary dependencies.
  - We recommend this method of installation so dependencies will also be automatically installed, and the Nuget package manager will show when updates are available.
1. It is also possible to Or manually download the packages from [Nuget](https://www.nuget.org/packages?q=thycotic "https://www.nuget.org/packages?q=thycotic"), but this requires more effort and you won't be notified of updates.
  1. The following packages are needed:
    - Thycotic.SecretServer.Sdk
	- Thycotic.SecretServer.Sdk.Extensions.Configuration
	- Thycotic.SecretServer.Sdk.Extensions.Integration
	- Thycotic.SecretServer.Sdk.Extensions.Integration.HttpModule
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
  and add the following:
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

New connection string with Secret Server SDK:

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

## SDK API

### SecretServerClient() Class

This class has the following methods:

#### .BustCache()

This method doesn't have an overload. Calling it destroys the SDK's cache of Secrets. The SDK configuration is still retained.

#### .Configure(IConfigSettings settings, [bool force = false])

<pre>
<code><strong>settings</strong>
Type: Object
Key value pairs to configure the SDK</code>
</pre>

<pre>
<code><strong>force (optional)</strong>
Type: boolean
Default: false
Forces the SDK to reconfigure itself</code>
</pre>

#### .GetSecret(int id)

This method returns a Secret object based on the REST secret model

<pre>
<code><strong>id</strong>
Type: int32
The secret id needed to retrieve the Secret</code>
</pre>

#### .GetSecretField(int id, string fieldslug)

This method gets a specific field from the Secret instead of returning the whole object

<pre>
<code><strong>id</strong>
Type: int32
The Secret Id needed to retieve the Secret</code>
</pre>

<pre>
<code><strong>fieldslug</strong>
Type: String
Slug identifier for the Secret field e.g. password</code>
</pre>

#### .GetAccessToken()

This method returns a REST API token from Secret Server, after authenticating with the configured SDK account.

#### .GetAccessTokenAsync()

Just like GetAccessToken but asynchronous.

## SDK Client

To get the command-line interface SDK Client tool check out our knowledge base article, [SDK Downloads](https://thycotic.force.com/support/s/article/SS-SDK-Downloads "https://thycotic.force.com/support/s/article/SS-SDK-Downloads").  This tool is not required to use the SDK NuGet packages.

The SDK Client and SDK NuGet packages are two separate projects with separate versioning.  We actively work to maintain feature parity between the two projects, but at times their features may differ.


