# Extending the CLI to new NLU providers

By default, the NLU.DevOps CLI tool supports [LUIS](https://www.luis.ai), [Lex](https://aws.amazon.com/lex/), and [Dialogflow](https://dialogflow.com). You can extend CLI support to additional NLU services by implementing the `INLUClientFactory` interface. In this guide, we'll walk through the creation of a very simple NLU service that stores training utterances and returns them when there is an exact match on the utterance text.

## Building the extension

Before starting, decide what the service identifier will be for you NLU service implementation, as that will be used in a number of places. E.g., the service identifier for LUIS is `luis`. In this example, we'll use `mock`.

### 1. Create a new .NET Core class library:
```bash
dotnet new console dotnet-nlu-mock
```

Be sure to replace `mock` with the service identifier you plan to use for your NLU service implementation.

### 2. Add required dependencies:
```bash
cd dotnet-nlu-mock
dotnet add package Newtonsoft.Json
dotnet add package NLU.DevOps.Core
dotnet add package System.Composition.AttributedModel
```

### 3. Implement `INLUService`
Open the project in [Visual Studio](https://visualstudio.microsoft.com/downloads/) or [Visual Studio Code](https://code.visualstudio.com/).

Add `MockNLUClient.cs` to your project:
```cs
// Copyright (c) Microsoft Corporation.
// Licensed under the MIT License.

namespace NLU.DevOps.MockProvider
{
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Threading;
    using System.Threading.Tasks;
    using Core;
    using Models;
    using Newtonsoft.Json;

    internal class MockNLUClient : DefaultNLUTestClient, INLUTrainClient
    {
        public MockNLUClient(string trainedUtterances)
        {
            this.Utterances = new List<LabeledUtterance>();
            if (trainedUtterances != null)
            {
                this.Utterances.AddRange(
                    JsonConvert.DeserializeObject<IEnumerable<LabeledUtterance>>(trainedUtterances));
            }
        }

        public string TrainedUtterances => JsonConvert.SerializeObject(this.Utterances);

        private List<LabeledUtterance> Utterances { get; }

        public Task TrainAsync(IEnumerable<LabeledUtterance> utterances, CancellationToken cancellationToken)
        {
            Utterances.AddRange(utterances);
            return Task.CompletedTask;
        }

        public Task CleanupAsync(CancellationToken cancellationToken)
        {
            return Task.CompletedTask;
        }

        protected override Task<LabeledUtterance> TestAsync(string utterance, CancellationToken cancellationToken)
        {
            var matchedUtterance = Utterances.FirstOrDefault(u => u.Text == utterance);
            return Task.FromResult(matchedUtterance ?? new LabeledUtterance(null, null, null));
        }

        protected override Task<LabeledUtterance> TestSpeechAsync(string speechFile, CancellationToken cancellationToken)
        {
            throw new NotImplementedException();
        }

        protected override void Dispose(bool disposing)
        {
        }
    }
}
```

### 4. Implement `INLUClientFactory`
Add `MockNLUClientFactory.cs` to your project:
```cs
// Copyright (c) Microsoft Corporation.
// Licensed under the MIT License.

namespace NLU.DevOps.MockProvider
{
    using System.Composition;
    using Microsoft.Extensions.Configuration;
    using Models;

    [Export("mock", typeof(INLUClientFactory))]
    public class MockNLUClientFactory : INLUClientFactory
    {
        public INLUTrainClient CreateTrainInstance(IConfiguration configuration, string settingsPath)
        {
            return new MockNLUClient(configuration["trainedUtterances"]);
        }

        public INLUTestClient CreateTestInstance(IConfiguration configuration, string settingsPath)
        {
            return new MockNLUClient(configuration["trainedUtterances"]);
        }
    }
}
```

Ensure that you set the `ExportAttribute` on the class with the service identifier you plan to use.

## Installing the extension

There are two ways to install the NLU service extension. You can specify a search root to the DLL for your service extension using the CLI tool, or you can install your project as a .NET Core tool extension on the same path that `dotnet-nlu` was installed.

### Specifying the include path

Build your .NET Core project:
```bash
dotnet build
```

When you want to run an NLU.DevOps CLI command, add the `--include` option with the path to your assembly:
```bash
dotnet nlu train --service mock --utterances utterances --include ./bin/Debug/netcoreapp2.1/dotnet-nlu-mock.dll
```

These commands assume you are currently in the `dotnet-nlu-mock` project folder.

### Installing via .NET Core tools

In order to install your NLU service implementation so it's always accessible to the NLU.DevOps CLI tool, you'll have to pack and install your project as a .NET Core tool extension.

To start, add configuration to your `dotnet-nlu-mock.csproj` file to instruct .NET Core to build your project as a .NET Core tool:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <AssemblyName>dotnet-nlu-mock</AssemblyName>
    <PackAsTool>true</PackAsTool> <!-- Add this line -->
  </PropertyGroup>
  ...
</Project>
```

Create a NuGet package from your project:
```bash
dotnet pack
```

Install the NuGet package as a .NET Core tool:
```bash
dotnet tool install dotnet-nlu-mock --add-source ./bin/Debug [-g|--tool-path <path>]
```

With this option, you won't have to specify the `--include` option each time you call the NLU.DevOps CLI tool.

### Example

Try running the following commands to install and test the `mock` service we've created above:
```
dotnet tool install -g dotnet-nlu
dotnet tool install -g dotnet-nlu-mock
dotnet nlu train -s mock -u models/utterances.json -a
dotnet nlu test -s mock -u models/tests.json
dotnet nlu clean -s mock -a
```
