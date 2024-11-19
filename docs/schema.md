---
sidebar_label: 'Schema'
---
# Schema Definition

The configuration technology utilized by SMA products leverages the ReactJS library [`react-jsonschema-form`](https://github.com/rjsf-team/react-jsonschema-form) in order to dynamically generate configuration forms from [**JSON Schema**](https://json-schema.org/) objects. In order to do this, two JSON Schema objects are required - a `data` object which describes the shape and types of the configuration object and a `ui-schema` object which describes how the generated form should be rendered. Both objects must be provided by invocations of appropriate methods in the ISchemaGenerator implementation.

In general when defining the schema information, all information required to complete a connection to the remote system should be defined within the IntegrationSchemaGenerator while information about executing a task should be defined within the TaskSchemaGenerator. In the AsyscoAMT implementation, the connection information base url, connection user and password are defined within the IntegrationSchemaGenerator module. 

Create a folder **Generators**.
In this folder create the **SchemaGenerator**, **IntegrationSchemaGenerator**, **TaskSchemaProtocols**, **TypeBatchJobSchemaGenerator** and **TypeScriptSchemaGenerator** classes.

![Project Structure](../static/img/project-structure.png)

## SchemaGenerator

The SchemaGenerator class is used to call the SDK SchemaGenerator using the schema definitions of the agent and task definitions.

```
using System.Globalization;

namespace AsyscoAMT.Generators;


public class SchemaGenerator(IConfig Config, ILogger Logger) : ISchemaGenerator
{
    public Task<SchemaResult> GetIntegrationSchema(CultureInfo? locale = null) => Task.FromResult(IntegrationSchemaGenerator.Generate());

   public IEnumerable<string> GetSubTypes() => TaskSchemaGenerator.Types;

   public Task<SchemaResult> GetTaskSchemaForType(ITaskConfig taskConfig, CultureInfo? locale = null)
   {
       return TaskSchemaGenerator.GetTaskSchemaForType(taskConfig);
   }
}
                
```
The **SchemaGenerator** class must implement the **ISchemaGenerator** interface. The class constructor should include the **IConfig Config** and **ILogger Logger** interfaces.
The SchemaGenerator class references two schema classes (one for the ACS Agent definition and one for the ACS Agent Task definition).

The above implementation shows how it is possible to have multiple task subTypes (see ISchemaGenerator section of document architecture for more information). 
In the above code, a reference is made to ***TaskSchemaGeneratorTypes*** which is a list of subtypes that the implementation supports. 
The code also provides a reference to both the created IntegrationSchemaGenerator and the TaskSchemaGenerator. 

## IntegrationSchemaGenerator

The IntegrationSchemaGenerator class defines the fields required for the ACS agent definition. THe class constructor should include the **IIntegration Integration** interface.

```
using ACSSDK.Implementation;
using ACSSDK.Models;
using AsyscoAmt;
using AsyscoAMT.Util;
using Newtonsoft.Json;

namespace AsyscoAMT.Generators;

public static class IntegrationSchemaGenerator
{
    public static SchemaResult Generate()
    {
        var schema = new ObjectField("Asysco AMT");
        schema.AddProperty("url", new StringField("Batch Server URL", "AsyscoAMT Batch Server URL"), true);
        schema.AddProperty("apiUser", new StringField("API User", "AsyscoAMT API User"), true);
        schema.AddProperty("apiUserPassword", new StringField("API User Password", "AsyscoAMT API User password"), true);

        var settings = new JsonSerializerSettings() { NullValueHandling = NullValueHandling.Ignore };
        var schemaStr = JsonConvert.SerializeObject(schema, Formatting.None, settings);

        var uiSchema = new
        {
            url = new { },
            apiUser = new { },
            apiUserPassword = new { }
        };
        var uiSchemaStr = JsonConvert.SerializeObject(uiSchema, Formatting.None);

        var schemaBuilder = new SchemaResultBuilder(new(schemaStr, uiSchemaStr, "{}", false), IntegrationFactory.AppName);
        return schemaBuilder.Build();

    }
}

```

In the above code snippet, three fields are defined consisting url, apiUser and apiUserPassword fields which are all required fields.
The above schema definition adds fields to the Solution Manager agent definition section.


![AsyscoAMT Agent definition fields](../static/img/acs-agent-fields.png)

## TaskSchemaGenerator

The TaskSchemaGenerator class defines the fields required for the AsyscoAMT Task definitions. The class constructor should include the **IConfig integrationConfig** and 
**ITaskConfig taskConfig** interfaces.

It includes the ***GetTaskSchemaForType*** method called by the SchemaGenerator class.

The Task definition screen is broken into three sections, the first section general arguments and the next two sections are used depending if a run job or a run script 
is being defined. Each section is defined by creating an ObjectField that groups the arguments.  

```
using ACSSDK.Implementation;
using ACSSDK.Interfaces;
using ACSSDK.Models;
using AsyscoAmt;
using AsyscoAMT.Util;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace AsyscoAMT.Generators;

public static class TaskSchemaGenerator
{

    private const string BatchJobType = "Batch Job";
    private const string ScriptType = "Script";

    public static readonly string[] Types = [BatchJobType, ScriptType];

    public static Task<SchemaResult> GetTaskSchemaForType(ITaskConfig taskConfig)
    {
        var schema = taskConfig.TaskType switch
        {
            BatchJobType => TypeBatchJobSchemaGenerator.Generate(taskConfig),
            ScriptType => TypeScriptSchemaGenerator.Generate(taskConfig),
            _ => new("{}", "{}", "{}", true)

        };

        return Task.FromResult(schema);
    }
}

```
The TaskSchemaGenerator includes a reference for each defined subtypes. 
In the above code there are two subtypes **BatchJobType** and **ScriptType**.
Each module is called in sequence to generate the schema for the task definition requirements.

The module TypeBatchJobSchemaGenerator includes the task definitions for an AsyscoAMT Batch Job.

![AsyscoAMT Batch Job Task definition fields](../static/img/acs-batch-job-task-fields.png)

```
using ACSSDK.Implementation;
using ACSSDK.Interfaces;
using ACSSDK.Models;
using AsyscoAmt;
using AsyscoAMT.Util;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace AsyscoAMT.Generators;

public static class TypeBatchJobSchemaGenerator
{

    public const string JobType = "Batch Job";

    public static SchemaResult Generate(ITaskConfig taskConfig)
    {
        var schema = GetDataSchema(taskConfig);
        var settings = new JsonSerializerSettings() { NullValueHandling = NullValueHandling.Ignore };
        var schemaStr = JsonConvert.SerializeObject(schema, Formatting.None, settings);
        var uiSchemaStr = GetUiSchema(taskConfig);

        var schemaBuilder = new SchemaResultBuilder(new(schemaStr, uiSchemaStr, "{}", false), IntegrationFactory.AppName);
        return schemaBuilder.Build();
    }
    private static ObjectField GetDataSchema(ITaskConfig taskConfig)
    {

        var schema = new ObjectField(JobType);

        schema.AddProperty("applicationName", new StringField("Application Name", "The name of the Application to be processed"), true);
        schema.AddProperty("amtSubmitUser", new StringField("Submit User", "Identifies the User that submitted the request - default BATCH").WithDefault("BATCH"), true);
        schema.AddProperty("amtUser", new StringField("User", "The AMT User to execute the job under (Run AS)"), false);
        schema.AddProperty("amtStation", new StringField("Station", "Identifies who submitted the request - default OPCON").WithDefault("OPCON"), true);
        schema.AddProperty("amtQueueName", new StringField("Queue Name", "(Optional) defines the AMT queue to place the processing request on"), false);
        schema.AddProperty("jobName", new StringField("Job Name", "The name of the Job in the Batch Server to execute"), true);
        schema.AddProperty("taskValues", new ArrayField("Task Values", new StringField("Values")).WithMaxItems(20));

        return schema;

    }

    private static string GetUiSchema(ITaskConfig config)
    {
        var autoPopulate = config.Config.autoPopulate;
        var disableAutoPopulated = autoPopulate is not null && (bool)autoPopulate.useAutoPopulate && (bool)autoPopulate.useAutoOverWrite;

        var uiSchema = new Dictionary<string, dynamic>
        {
            ["ui:order"] = new List<string>
            {
                "applicationName", // required
                "amtSubmitUser", // required
                "amtUser",
                "amtStation", // required
                "amtQueueName",
                "jobName", // required
                "taskValues"
            }
        };

        var uiSchemaStr = JsonConvert.SerializeObject(uiSchema, Formatting.None);
        return uiSchemaStr;
    }

}

```
The above codes shows how to implement a list for the **taskValues** 

The module TypeScriptSchemaGenerator includes the task definitions for an AsyscoAMT Batch Job.

![AsyscoAMT Script Task definition fields](../static/img/acs-script-task-fields.png)

```
using ACSSDK.Implementation;
using ACSSDK.Interfaces;
using ACSSDK.Models;
using AsyscoAMT.Util;
using AsyscoAmt;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace AsyscoAMT.Generators;

public static class TypeScriptSchemaGenerator
{
    public const string JobType = "Script";

    public static SchemaResult Generate(ITaskConfig taskConfig)
    {
        var schema = GetDataSchema(taskConfig);
        var settings = new JsonSerializerSettings() { NullValueHandling = NullValueHandling.Ignore };
        var schemaStr = JsonConvert.SerializeObject(schema, Formatting.None, settings);
        var uiSchemaStr = GetUiSchema(taskConfig);

        var schemaBuilder = new SchemaResultBuilder(new(schemaStr, uiSchemaStr, "{}", false), IntegrationFactory.AppName);
        return schemaBuilder.Build();
    }
    private static ObjectField GetDataSchema(ITaskConfig taskConfig)
    {

        var schema = new ObjectField(JobType);

        schema.AddProperty("applicationName", new StringField("Application Name", "The name of the Application to be processed"), true);
        schema.AddProperty("amtSubmitUser", new StringField("Submit User", "Identifies the User that submitted the request - default BATCH").WithDefault("BATCH"), true);
        schema.AddProperty("amtUser", new StringField("User", "The AMT User to execute the job under (Run AS)"), false);
        schema.AddProperty("amtStation", new StringField("Station", "Identifies who submitted the request - default OPCON").WithDefault("OPCON"), true);
        schema.AddProperty("amtQueueName", new StringField("Queue Name", "(Optional) defines the AMT queue to place the processing request on"), false);
        schema.AddProperty("scriptName", new StringField("Script Name", "The name of the script file to execute"), true);
        schema.AddProperty("scriptParameters", new ArrayField("Script Parameters", new StringField("Parameter")).WithMaxItems(20));

        return schema;

    }

    private static string GetUiSchema(ITaskConfig config)
    {
        var autoPopulate = config.Config.autoPopulate;
        var disableAutoPopulated = autoPopulate is not null && (bool)autoPopulate.useAutoPopulate && (bool)autoPopulate.useAutoOverWrite;

        var uiSchema = new Dictionary<string, dynamic>
        {
            ["ui:order"] = new List<string>
            {
                "applicationName", // required
                "amtSubmitUser", // required
                "amtUser",
                "amtStation", // required
                "amtQueueName",
                "scriptName", // required
                "scriptParameters"
            }
        };

        var uiSchemaStr = JsonConvert.SerializeObject(uiSchema, Formatting.None);
        return uiSchemaStr;
    }

}

```
The above codes shows how to implement a list for the **ScriptParameters** 

## SchemaGeneratorUtils
This is a utility class contained in the **Util** directory and contains various methods to assist with schema generatio.

```
using Newtonsoft.Json.Serialization;
using Newtonsoft.Json;

namespace AsyscoAMT.Util;

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class ObjectField(string? Title = null, string? Description = null)
{
    public string? Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "object";
    public Dictionary<string, dynamic> Properties = [];
    public List<string> Required = [];
    public Dictionary<string, dynamic> Definitions = [];
    public List<dynamic> AllOf = [];

    public void AddProperty(string key, dynamic value, bool required = false)
    {
        Properties.Add(key, value);
        if (required) Required.Add(key);
    }

    public ObjectField WithConditional<T>(string propertyName, T propertyValue, ObjectField ifTrue)
    {
        var condition = new Condition<T>(propertyValue);
        var conditionClause = new ConditionClause<T>(new() { { propertyName, condition } });
        AllOf.Add(new AllOfConditional<T>(conditionClause, ifTrue));
        return this;
    }

    public ObjectField WithDefinition(string name, dynamic definition)
    {
        Definitions.Add(name, definition);
        return this;
    }

    public bool ShouldSerializeProperties() => Properties.Count > 0;
    public bool ShouldSerializeRequired() => Required.Count > 0;
    public bool ShouldSerializeDefinitions() => Definitions.Count > 0;
    public bool ShouldSerializeAllOf() => AllOf.Count > 0;
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class ArrayField(string Title, dynamic ItemTemplate, string? Description = null)
{
    public string Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "array";
    public int? MaxItems { get; set; } = null;
    public dynamic Items { get; init; } = ItemTemplate;
    public bool? UniqueItems { get; set; }

    public ArrayField WithMaxItems(int count)
    {
        MaxItems = count;
        return this;
    }

    public ArrayField WithUniqueItems(bool uniqueItems = true)
    {
        UniqueItems = uniqueItems;
        return this;
    }
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class NullField(string? Title, string? Description = null)
{
    public string? Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "null";
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class StringField(string Title, string? Description = null)
{
    public string Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "string";
    public int? MaxLength { get; set; } = null;
    public int? MinLength { get; set; } = null;
    public string Default { get; set; } = null;

    public StringField WithMaxLength(int len)
    {
        MaxLength = len;
        return this;
    }

    public StringField WithMinLength(int len)
    {
        MinLength = len;
        return this;
    }

    public StringField WithDefault(string defaultValue)
    {
        Default = defaultValue;
        return this;
    }
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class IntField(string Title, string? Description = null)
{
    public string Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "integer";
    public int? Maximum { get; set; } = null;
    public int? Minimum { get; set; } = null;
    public int? Default { get; set; } = null;

    public IntField WithMax(int val)
    {
        Maximum = val;
        return this;
    }

    public IntField WithMin(int val)
    {
        Minimum = val;
        return this;
    }

    public IntField WithDefault(int val)
    {
        Default = val;
        return this;
    }
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class BoolField(string Title, string? Description = null)
{
    public string Title { get; init; } = Title;
    public string? Description { get; init; } = Description;
    public string Type = "boolean";
    public bool Default { get; set; } = false;

    public BoolField WithDefault(bool val)
    {
        Default = val;
        return this;
    }
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class RefField(string Title, string Ref)
{
    public string Title { get; init; } = Title;

    [JsonProperty("$ref")]
    public string Ref { get; init; } = Ref;
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class EnumEntry(string Entry, string Type)
{
    public string Type = Type;
    public string Title = Entry;
    public List<string> Enum = [Entry];
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class EnumField(string Title, string Type, IEnumerable<string> Values)
{

    public string Title { get; init; } = Title;

    public string Type { get; init; } = Type;

    public string Default { get; init; } = Values.FirstOrDefault(string.Empty);

    public IEnumerable<EnumEntry> AnyOf { get; init; } = Values.Select(v => new EnumEntry(v, Type));
}

public class Condition<T>(T value)
{
    public T Const { get; init; } = value;
}

public class ConditionClause<T>(Dictionary<string, Condition<T>> Properties)
{
    public Dictionary<string, Condition<T>> Properties { get; init; } = Properties;
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class AllOfConditional<T>(ConditionClause<T> If, ObjectField Then)
{

    public ConditionClause<T> If { get; init; } = If;
    public ObjectField Then { get; init; } = Then;
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class ObjectEnumEntry<T>(Func<T, string> getNameFunc, IEnumerable<T> options)
{
    public IEnumerable<string> EnumNames { get; set; } = options.Select(getNameFunc).ToList();
    public IEnumerable<T> Enum { get; set; } = options;
}

[JsonObject(NamingStrategyType = typeof(CamelCaseNamingStrategy))]
public class ObjectEnumField<T>(string? title, Func<T, string> getNameFunc, IEnumerable<T> options)
{
    public string? Title { get; set; } = title;
    public string Type { get; } = "array";
    public bool UniqueItems { get; } = true;
    public ObjectEnumEntry<T> Items { get; } = new(getNameFunc, options);
}

```
                    SchemaGeneratorUtils.cs 