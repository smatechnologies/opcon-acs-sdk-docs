---
sidebar_label: 'Develop'
---

# Developing ACS Solution

The ACS implementation is a C# .net Library. 

The SDK includes 2 sample implementations, ElementaryIntegration and OpConIntegration. Each sample provides
different approaches to creating integrations.

The SDK documentation is based on the AsyscoAMT implementation.

- ElementaryIntegration shows various schema definitions for creating tasks definition screens.
- OpConIntegration shows an implementation of selecting and running tasks in a remote opCon system.

1. Create a C# Library project.
2. Add Nuget packages ACSSDK (SMA Package).
3. Add the initial **IntegrationFactory** and **Integration** classes.

![Project Structure](../static/img/project-structure.png)

The above diagram shows a typical project structure that will be used during the remainder of the SDK documentation.

## Create IntegrationFactory class
At the root level create the **IntegrationFactory** class and insert the following code.

```
using ACSSDK.Interfaces;
using AsyscoAMT;
using Microsoft.Extensions.Logging;
using System.Globalization;

namespace AsyscoAmt;

public class IntegrationFactory : IIntegrationFactory
{
    public const string AppName = "AsyscoAMT";

    public IEnumerable<string> SupportedApplications { get; } = [AppName];

    public IEnumerable<CultureInfo> SupportedLanguages => [new("en-US")];

    public string Version => "0.0.1";

    public IIntegration CreateIntegration(string application, IConfig integrationInfo, ILogger logger)
    {
        if (application != AppName)
        {
            throw new ArgumentException($"Unsupported application {application}", nameof(application));
        }

        return new Integration(integrationInfo, logger);
    }
}

```
                    IntegrationFactory.cs 

The **IntegrationFactory** class must implement the **IIntegrationFactory** interface.

It provides the information about the ACS implementation and is the entry point into the ACS development.

## Create Integration class
At the root level create the **Integration** class and insert the following code.

```
using ACSSDK.Interfaces;
using AsyscoAMT.Generators;
using AsyscoAMT.Protocols;
using Microsoft.Extensions.Logging;

namespace AsyscoAMT;

public sealed class Integration : IIntegration
{

    public IConfig IntegrationInfo { get; set; }

    public IIntegrationProtocol IntegrationProtocol { get; init; }

    public ITaskLogProtocol TaskLogProtocol { get; init; }

    public ISchemaGenerator SchemaGenerator { get; init; }

    public ITaskProtocol TaskProtocol { get; init; }

    public Integration(IConfig config, ILogger logger)
    {
        IntegrationInfo = config;
        IntegrationProtocol = new IntegrationProtocol(logger);
        SchemaGenerator = new SchemaGenerator(IntegrationInfo, logger);
        TaskProtocol = new TaskProtocol(this, logger);
        TaskLogProtocol = new TaskLogProtocol(this, logger);
    }

    private bool disposedValue;

    private void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
                // TODO: dispose managed state (managed objects)
            }

            disposedValue = true;
        }
    }
    public void Dispose()
    {
        // Do not change this code. Put cleanup code in 'Dispose(bool disposing)' method
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}

```
                    Integration.cs 

The **Integration** class must implement the **IIntegration** interface.

The Integration class sets up the references to the following modules

- IntegrationInfo       class supplied by the ACSSDK.
- IntegrationProtocol   class to be created that manages the connection between the ACS implementation and the target system.
- TaskLogProtocol       class to be created that retrieves joblogs from the target system used by the OpCon JORS function.
- SchemaGenerator       class to be created that defines the json schemas that contain the agent and task configuration requirements.
- TaskProtocol          class to be created that defines how tasks on the target system are managed.  

Includes the **Integration** method called by the IntegrationFactory.