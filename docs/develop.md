---
slug: '/'
sidebar_label: 'Develop'
---

# Developing ACS Solution

The ACS implementation is a C# .net Library. 

The SDK includes 2 sample implementations, ElementaryIntegration and OpConIntegration.

- ElementaryIntegration shows various schema definitions for creating tasks definition screens.
- OpConIntegration shows an implementation of selecting and running tasks in a remote opCon system.

1. Create a C# Library project.
2. Add Nuget packages ACSSDK (SMA Package).
3. Add Nuget Newtonsoft.Json package.
4. Add the initial **IntegrationFactory** and **Integration** classes.

## Create IntegrationFactory class
At the root level create the **IntegrationFactory** class and insert the following code.

```
using ACSSDK.Interfaces;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Reflection;
using System.Text;
using System.Threading.Tasks;

namespace ACSTest;

public class IntegrationFactory : IIntegrationFactory
{
    public const string ACSTEST = "ACSTest";

    //   public string Version => "0.1.0";
    public string Version => Assembly.GetAssembly(GetType())?.GetName().Version?.ToString() ?? "0.1.0";

    public IEnumerable<string> SupportedApplications => new List<string> {ACSTEST};

    public IEnumerable<CultureInfo> SupportedLanguages => new List<CultureInfo> { new("en-US") };

    public IIntegration CreateIntegration(string application, IConfig config, ILogger logger) => application switch
    {
        ACSTEST => new Integration(config, logger),
        _ => throw new NotImplementedException(nameof(application))
    };
}

```
The class must implement the **IIntegrationFactory** interface.

Provide a unique name and version for your ACS implementation (in the above example the name is ACSTest and  the version is 0.1.0). 

## Create Integration class
At the root level create the **Integration** class and insert the following code.

```
using ACSSDK.Models;
using ACSSDK.Interfaces;
using Microsoft.Extensions.Logging;
using ACCSTest.SchemaGenerators;
using ACCSTest.Protocols;

namespace ACSTest;

public sealed class Integration : IIntegration
{
    public IConfig IntegrationInfo { get; init; }
    public IIntegrationProtocol IntegrationProtocol { get; init; }

    public ITaskLogProtocol TaskLogProtocol { get; init; }

    public ISchemaGenerator SchemaGenerator { get; init; }

    public ITaskProtocol TaskProtocol { get; init; }

    public Integration(IConfig config, ILogger logger)
    {
        IntegrationInfo = config;
        IntegrationProtocol = new IntegrationProtocol(logger);
        TaskProtocol = new TaskProtocol(this, logger);
        TaskLogProtocol = new TaskLogProtocol(this, logger);
        SchemaGenerator = new SchemaGenerator(this, logger);

        logger.LogInformation("Loading ACSTest Config version {v}", config.Version);
    }

    public void Dispose() => GC.SuppressFinalize(this);
}

```
The class must implement the **IIntegration** interface.

The Integration class sets up the references to the following modules

- IntegrationInfo       class supplied by the ACSSDK.
- IntegrationProtocol   class to be created that manages the connection between the ACS implementation and the target system.
- TaskLogProtocol       class to be created that retrieves joblogs from the target system used by the OpCon JORS function.
- SchemaGenerator       class to be created that defines the json schemas that contain the agent and task configuration requirements.
- TaskProtocol          class to be created that defines how tasks on the target system are managed.  

