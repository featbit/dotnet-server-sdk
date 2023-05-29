# FeatBit Server-Side SDK for .NET

## Introduction

This is the .NET Server-Side SDK for the 100% open-source feature flags management
platform [FeatBit](https://github.com/featbit/featbit).

The FeatBit Server-Side SDK for .NET is designed primarily for use in multi-user systems such as web servers and
applications. It is not intended for use in desktop and embedded systems applications.

## Data Synchronization

We use websocket to make the local data synchronized with the FeatBit server, and then store them in memory by
default. Whenever there is any change to a feature flag or its related data, this change will be pushed to the SDK and
the average synchronization time is less than 100 ms. Be aware the websocket connection may be interrupted due to
internet outage, but it will be resumed automatically once the problem is gone.

If you want to use your own data source, see [Offline Mode](#offline-mode).

## Get Started

### Installation

The latest stable version is available on [NuGet](https://www.nuget.org/packages/FeatBit.ServerSdk/).

```sh
dotnet add package FeatBit.ServerSdk
```

Use the `--version` option to specify
a [preview version](https://www.nuget.org/packages/FeatBit.ServerSdk/absoluteLatest) to install.

### Quick Start

The following code demonstrates basic usage of FeatBit.ServerSdk.

```cs
using FeatBit.Sdk.Server;
using FeatBit.Sdk.Server.Model;

// Set secret to your FeatBit SDK secret.
const string secret = "<replace-with-your-env-secret>";

// Creates a new client instance that connects to FeatBit with the default option.
var client = new FbClient(secret);
if (!client.Initialized)
{
    Console.WriteLine("FbClient failed to initialize. All Variation calls will use fallback value.");
}
else
{
    Console.WriteLine("FbClient successfully initialized!");
}

// flag to be evaluated
const string flagKey = "game-runner";

// create a user
var user = FbUser.Builder("anonymous").Build();

// evaluate a boolean flag for a given user
var boolVariation = client.BoolVariation(flagKey, user, defaultValue: false);
Console.WriteLine($"flag '{flagKey}' returns {boolVariation} for user {user.Key}");

// evaluate a boolean flag for a given user with evaluation detail
var boolVariationDetail = client.BoolVariationDetail(flagKey, user, defaultValue: false);
Console.WriteLine(
    $"flag '{flagKey}' returns {boolVariationDetail.Value} for user {user.Key}. " +
    $"Reason Kind: {boolVariationDetail.Kind}, Reason Description: {boolVariationDetail.Reason}"
);

// close the client to ensure that all insights are sent out before the app exits
await client.CloseAsync();
```

### Examples

- [Console App](/examples/ConsoleApp/Program.cs)
- [ASP.NET Core](/examples/WebApiApp/Program.cs)

## SDK

### FbClient

The FbClient is the heart of the SDK which providing access to FeatBit server. Applications should instantiate a **single instance** for the lifetime of the application.

#### FbClient Using Default Options

```csharp
using FeatBit.Sdk.Server;

// Creates a new client instance that connects to FeatBit with the default option.
var client = new FbClient("<replace-with-your-env-secret>");
```

#### FbClient Using Custom Options

```csharp
using FeatBit.Sdk.Server;
using FeatBit.Sdk.Server.Options;
using Microsoft.Extensions.Logging;

var consoleLoggerFactory = LoggerFactory.Create(x => x.AddConsole());

var options = new FbOptionsBuilder("<replace-with-your-env-secret>")
    .Steaming(new Uri("ws://localhost:5100"))
    .Event(new Uri("http://localhost:5100"))
    .StartWaitTime(TimeSpan.FromSeconds(3))
    .LoggerFactory(consoleLoggerFactory)
    .Build();

// Creates a new client instance that connects to FeatBit with the custom option.
var client = new FbClient(options);
```

### FbUser

FbUser defines the attributes of a user for whom you are evaluating feature flags. FbUser has two built-in
attributes: `key` and `name`. The only mandatory attribute of a FbUser is the key, which must uniquely identify each
user.

Besides these built-in properties, you can define any additional attributes associated with the user
using `Custom(string key, string value)` method on `IFbUserBuilder`. Both built-in attributes and custom attributes can
be referenced in targeting rules, and are included in analytics data.

There is only one method for building FbUser.

```csharp
var bob = FbUser.Builder("a-unique-key-of-user")
    .Name("bob")
    .Custom("age", "15")
    .Custom("country", "FR")
    .Build();
```

### Evaluating flags

By using the feature flag data it has already received, the SDK **locally calculates** the value of a feature flag for a
given user.

There is a `Variation` method that returns a flag value, and a `VariationDetail` method that returns an object
describing how the value was determined for each type.

- BoolVariation/BoolVariationDetail
- StringVariation/StringVariationDetail
- DoubleVariation/DoubleVariationDetail
- FloatVariation/FloatVariationDetail
- IntVariation/IntVariationDetail
- JsonVariation/JsonVariationDetail (in consideration)

Variation calls take the feature flag key, a FbUser, and a default value. If any error makes it impossible to
evaluate the flag (for instance, the feature flag key does not match any existing flag), default value is returned.

```csharp
using FeatBit.Sdk.Server;
using FeatBit.Sdk.Server.Model;

// Creates a new client instance that connects to FeatBit with the default option.
var client = new FbClient("<replace-with-your-env-secret>");

// The flag key to be evaluated
const string flagKey = "game-runner";

// The user
var user = FbUser.Builder("anonymous").Build();

// Evaluate a boolean flag for a given user
var boolVariation = client.BoolVariation(flagKey, user, defaultValue: false);
Console.WriteLine($"flag '{flagKey}' returns {boolVariation} for user {user.Key}");

// evaluate a boolean flag for a given user with evaluation detail
var boolVariationDetail = client.BoolVariationDetail(flagKey, user, defaultValue: false);
Console.WriteLine(
    $"flag '{flagKey}' returns {boolVariationDetail.Value} for user {user.Key}. " +
    $"Reason Kind: {boolVariationDetail.Kind}, Reason Description: {boolVariationDetail.Reason}"
);
```

### Offline Mode

In some situations, you might want to stop making remote calls to FeatBit. Here is how:

```csharp
var options = new FbOptionsBuilder()
    .Offline(true)
    .Build();

var client = new FbClient(options);
```

When you put the SDK in offline mode, no insight message is sent to the server and all feature flag evaluations return
fallback values because there are no feature flags or segments available. If you want to use your own data source in
this case, the sdk allows users to populate feature flags and segments data from a JSON string. Here is an
example: [featbit-bootstrap.json](/tests/FeatBit.ServerSdk.Tests/Bootstrapping/featbit-bootstrap.json).

> **_NOTE:_** Populating data from a JSON string is only supported in offline mode.

The format of the data in flags and segments is defined by FeatBit and is subject to change. Rather than trying to
construct these objects yourself, it's simpler to request existing flags directly from the FeatBit server in JSON format
and use this output as the starting point for your file. Here's how:

```shell
# replace http://localhost:5100 with your evaluation server url
curl -H "Authorization: <your-env-secret>" http://localhost:5100/api/public/sdk/server/latest-all > featbit-bootstrap.json
```

Then use that file to initialize FbClient:

```csharp
using FeatBit.Sdk.Server.Options;

var json = File.ReadAllText("featbit-bootstrap.json");

var options = new FbOptionsBuilder()
    .Offline(true)
    .UseJsonBootstrapProvider(json)
    .Build();

var client = new FbClient(options);
```

## Supported .NET versions

This version of the SDK is built for the following targets:

- .NET 6.0: runs on .NET 6.0 and above (including higher major versions).
- .NET Core 3.1: runs on .NET Core 3.1+.
- .NET Framework 4.6.2: runs on .NET Framework 4.6.2 and above.
- .NET Standard 2.0/2.1: runs in any project that is targeted to .NET Standard 2.x rather than to a specific runtime
  platform.

The .NET build tools should automatically load the most appropriate build of the SDK for whatever platform your
application or library is targeted to.

> **_NOTE:_** This SDK requires the `System.Text.Json` API to be available, which is included in the runtime for .NET
> Core 3.1 and later versions, but not on other platforms, so on other platforms the SDK brings
> in `System.Text.Json` as a NuGet package dependency.

## Getting support

- If you have a specific question about using this sdk, we encourage you
  to [ask it in our slack](https://join.slack.com/t/featbit/shared_invite/zt-1ew5e2vbb-x6Apan1xZOaYMnFzqZkGNQ).
- If you encounter a bug or would like to request a
  feature, [submit an issue](https://github.com/featbit/dotnet-server-sdk/issues/new).

## See Also

- [Connect To .NET Sdk](https://featbit.gitbook.io/docs/getting-started/4.-connect-an-sdk/server-side-sdks/.net-sdk)
