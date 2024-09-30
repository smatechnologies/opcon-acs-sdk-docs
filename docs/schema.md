---
slug: '/'
sidebar_label: 'Schema'
---
# Schema Definition

The configuration technology utilized by SMA products leverages the ReactJS library [`react-jsonschema-form`](https://github.com/rjsf-team/react-jsonschema-form) in order to dynamically generate configuration forms from [**JSON Schema**](https://json-schema.org/) objects. In order to do this, two JSON Schema objects are required - a `data` object which describes the shape and types of the configuration object and a `ui-schema` object which describes how the generated form should be rendered. Both objects must be provided by invocations of appropriate methods in the ISchemaGenerator implementation.

Create a folder **SchemaGenerators**.
In this folder create the **SchemaGenerator**, **IntegrationSchemaGenerator** and **TaskSchemaProtocols** classes.

## SchemaGenerator

The SchemaGenerator class is used to call the SDK SchemaGenerator using the schema definitions of the agent and task definitions.

```
sing ACSSDK.Interfaces;
using ACSSDK.Models;
using Microsoft.Extensions.Logging;
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using ACSSDK.Interfaces;
using ACSSDK.Models;
using Microsoft.Extensions.Logging;
using System.Globalization;

namespace ACSTest.SchemaGenerators;

public class SchemaGenerator(IIntegration Integration, ILogger Logger) : ISchemaGenerator
{

    private readonly IntegrationSchemaGenerator IntegrationSchemaGenerator = new(Integration);
    private readonly TaskSchemaGenerator TaskSchemaGenerator = new(Integration);

    public Task<SchemaResult> GetIntegrationSchema(CultureInfo? locale = null)
    {
        return IntegrationSchemaGenerator.GetIntegrationSchema();
    }

    public IEnumerable<string> GetSubTypes() => TaskSchemaGenerator.Types;

    public Task<SchemaResult> GetTaskSchemaForType(ITaskConfig taskConfig, CultureInfo? locale = null)
    {
        Logger.LogDebug("Loading Task Config version {v}", taskConfig.Version);
        Logger.LogDebug("Getting task schema with culture {c}", locale?.Name ?? "null");
        return TaskSchemaGenerator.GetTaskSchemaForType(taskConfig);
    }
}

```

The **SchemaGenerator** class must implement the **ISchemaGenerator** interface. The class constructor should include the **IIntegration Integration** and **ILogger Logger** interfaces.
The SchemaGenerator class references two schema classes (one for the ACS Agent definition and one for the ACS Agent Task definition).

The above implementation shows how it is possible to have multiple task subTypes (see ISchemaGenerator section of document architecture for more information). 
In the above code, the SchemaGenerator must reference both an IntegrationSchemaGenerator and a TaskSchemaGenerator. 

## IntegrationSchemaGenerator

The IntegrationSchemaGenerator class defines the fields required for the ACS agent definition.

```
using ACSSDK.Interfaces;
using ACSSDK.Models;
using Newtonsoft.Json.Linq;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace ACSTest.SchemaGenerators
{
    public class IntegrationSchemaGenerator(IIntegration Integration)
    {
        public const string KeyApiUrl = "apiUrl";
        public const string KeyApiUser = "apiUser";
        public const string KeyApiUserPassword = "apiUserPassword";
        private const string KeyMinLength = "minLength";

        private const string TitleACSTest = "ACSTest Integration Configuration";
        private const string TitleApiUrl = "Api Url for remote ACSTest system";
        private const string TitleApiUser = "Api User Name";
        private const string TitleApiUserPassword = "Api User Password";
        private const string DescriptionACSTest = "Agent configuration for remote ACSTest Server";

        public async Task<SchemaResult> GetIntegrationSchema()
        {

            var config = Integration.IntegrationInfo.Config;

            var requiredParams = new List<string>() { KeyApiUrl, KeyApiUser, KeyApiUserPassword };

            var keyApiUrl = new
            {
                title = TitleApiUrl,
                type = "string",
                minLength = 1,
            };

            var keyApiUser = new
            {
                title = TitleApiUser,
                type = "string",
                minLength = 1,
            };

            var keyApiUserPassword = new
            {
                title = TitleApiUserPassword,
                type = "string",
                minLength = 1,
            };

            var rootPropertiesDictionary = new Dictionary<string, dynamic>()
        {
            { KeyApiUrl, keyApiUrl },
            { KeyApiUser, keyApiUser },
            {KeyApiUserPassword, keyApiUserPassword }
        };

            var dataSchema = new
            {
                title = TitleACSTest,
                description = DescriptionACSTest,
                type = "object",
                properties = rootPropertiesDictionary,
                required = new JArray(requiredParams),
            };

            return new
            (
                JsonConvert.SerializeObject(dataSchema),
                UISchema: "{}",
                ExtraErrors: "{}",
                false
            );
        }
    }
}

```
In the above code snippet, three fields are defined consisting url, user and password fields which are required fields.
The above schema definition adds fields to the Solution Manager agent definition section.

![ACS Agent fields](../static/img/acs-agent-fields.png)