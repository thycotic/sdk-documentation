# Notes before using the integrated SDK

## You will need to install the following if not installed

- .NetStandard2.0 which equals to
  - .NET core 2.0 if you’re building a .NET core application
  - .NET (Full Framework) 4.6.1
- Create a rule in Secret Server for client onboarding:
  - Navigate to Admin > SDK Client Management > Client Onboarding Tab
  - Click on the “+ Rule” button to create a new Rule
  - Name your Rule (Something the helps identify what this is)
  - Assign IPv4 restrictions (optional)
  - Assign an Application User Account (create one if you hadn’t already)
  - Generate a Rule Key (optional)
  - Save
- The application account you created needs to have access to the Secrets your application needs. Ensure the permissions are accurate at the folder/secret level

## SDK Integration in C# project

- Download the SDK files from NuGet. You can do this either from NuGet.org or directly from Visual Studio
  - Search for Thycotic
  - Install the following packages in no order
  - Thycotic.SecretServer.SDK
  - Thycotic.SecretServer.SDK.Extension.Configuration
  - Thycotic.SecretServer.SDK.Extension.Integration
- In your Project add the following references:
  - using Thycotic.SecretServer.Sdk.Extensions.Integration.Clients;
  - using Thycotic.SecretServer.Sdk.Extensions.Integration.Models;
  - using Thycotic.SecretServer.Sdk.Infrastructure.Models;

Alternatively, you can instantiate the objects and let Visual Studio add the references for you.

## Configuring the SDK to connect to your Secret Server

You will have to instantiate a new SecretServerClient object to interact with the SDK. Below is a sample code on how to do that:

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
The code above will register the integrated client with Secret Server. Below is an explanation of the various methods and properties for that sample code:
- ConfigSettings is an object
- SecretServerUrl is the base URL for your Secret Server (if you have a Load Balancer then it should be the load balanced URL
- RuleName is the name of the rule we created earlier
- RuleKey is the Key we generated, can be NULL if no key
- CacheStrategy is how the SDK will cache requests. This is an enum that contains four values:
  - CacheThenServer
  - CacheThenServerAllowExpired (This will allow fallback in case Secret Server isn’t available and the cache is expired, the SDK will still use the cache until it can contact Secret Server and prime the cache)
  - Never
  - ServerThenCache (This mode is for redundancy since it will fall back to cache in case Secret Server isn’t available)
- CacheAge is how long the cache is valid before it expires in minutes
- ResetToken  is a random string to revoke clients and reinitialize them
Now we can use the client object to get a Secret, Token, or a Secret field from Secret Server, and feed that to our application
```c#

var password= client.GetSecretField(int,"password"); //replace int with Secret ID

```
The code above will retrieve a password from Secret Server, which we can then pass to a connection string or anywhere a password is needed

You can call the ```GetSecret();``` method on the client object to get the full Secret object, and then access the items property which holds a collection of the Secret fields and their values. How you access these values is up to you, but you can simple use Linq to query what you’re looking for e.g.
```C#

var secret = client.GetSecret(int); //replace int with a secret ID
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

so the whole sample code would look like this
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
