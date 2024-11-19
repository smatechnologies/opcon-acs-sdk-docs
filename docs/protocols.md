---
sidebar_label: 'Protocols'
---

# Developing ACS Protocols

The protocol classes implement the link between the ACS environment and the remote system. Separate protocol classes must be created for managing the communications link, managing tasks and retrieving task logs. 

![Project Structure](../static/img/project-structure.png)

Create a folder **Protocols**.
In this folder create the **IntegrationProtocol**, **TaskLogProtocol** and **TaskProtocol** classes.

## IntegrationProtocol

The **IntegrationProtocol** class must implement the **IIntegrationProtocol** interface. The class constructor should include the **ILogger Logger** interface that writes log messages to the **SMAApiAgentNetcom.log** file in the SAM/Log directory. 

```

using ACSSDK.Interfaces;
using ACSSDK.Models;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;
using System.Text.Json;
using System.Threading.Tasks;

namespace ACCTest.Protocols;

public class IntegrationProtocol(ILogger Logger) : IIntegrationProtocol
{

     public Task<Result> InitialStatus(IConfig integrationConfig)
    {
        // perform heartbeat check
        var url = integrationConfig.Config.url;
        var user = integrationConfig.Config.apiUser;
        var password = integrationConfig.Config.apiUserPassword;

        if (string.IsNullOrEmpty(url) || string.IsNullOrEmpty(user) || string.IsNullOrEmpty(password))
        {
            Logger.LogCritical("Communication can not be established : Invalid Configuration");
            return Task.FromResult(new Result(ResultCode.NotCommunicating));
        }
        Logger.LogCritical($"url {url} user {user} pword {password}");

        bool running = false;
        running = performHeartBeat(url).Result;
        if(running == true)
        {
            Logger.LogCritical("AsyscoAMT - Communications established");
            return Task.FromResult(new Result(ResultCode.Communicating));
        }
    else
        {
            Logger.LogCritical("Communication can not be established : No connection");
            return Task.FromResult(new Result(ResultCode.NotCommunicating));
        }
    }

    public Task<Result> Status(IConfig integrationConfig)
    {
        // perform login, logoff
        var url = integrationConfig.Config.url;
        var user = integrationConfig.Config.apiUser;
        var password = integrationConfig.Config.apiUserPassword;

        bool running = false;
        running = performHeartBeat(url).Result;
        if (running == true)
        {
            Logger.LogCritical("AsyscoAMT - Communications established");
            return Task.FromResult(new Result(ResultCode.Communicating));
        }
        else
        {
            Logger.LogCritical("Communication can not be established : No connection");
            return Task.FromResult(new Result(ResultCode.NotCommunicating));
        }
    }

    private async Task<bool> performHeartBeat(
        string url
        ) 
    {
        bool running = false;
        string? heartBeat = null;

        HttpClient client = new AsyscoAmtApi(Logger).GetClient(url, null);
        Logger.LogCritical($"Url : {url}");
        heartBeat = await new AsyscoAmtApi(Logger).HeartBeat(client, url);
        if (heartBeat is not null)
        {
            running = true;
        }
        else
        {
            running = false;
        }
        client.Dispose();
        return running;
    }
}

```

The IntegrationProtocol class manages the connection between the ACS implementation and the remote system. After configuration has been completed and the ACS implementation is marked-up, the **InitialStatus** method is called to establish the connection to the remote system. If the connection is successfully established, a **Communicating** status is returned or if unsuccessful a **NotCommunicating** status is returned. If a **Communicating** status is returned, the ACS agent will be marked-up and it will be possible to submit task requests to the remote system.

Once the connection is established, the **Status** method is called on a regular basis to determine if the remote system is still available top process task requests. If available a **Communicating** status is returned, if not available a **NotCommunicating** status is returned and the ACS agent will be marked-down and no tasks can be started.    

The class must contain the **InitialStatus** and **Status** methods. In each case the methods accept the **IConfig** interface which provides the ACS agent configuration information.

The above code snippet uses the **performHeartBeat** to send a Rest-API request to the Asysco AMT Batch server to determine if the batch server is available to execute tasks. 

The inclusion of **ILogger Logger** allows logging messages to be displayed in the SMAApiAgentNetcom.log file. 

## TaskProtocol

The **TaskProtocol** class must implement the **ITaskProtocol** interface. The class constructor should include the **IIntegration Intgration** and **ILogger Logger** interfaces that provides access to the agent information and that writes log messages to the **SMAApiAgentNetcom.log** file in the SAM/Log directory. 

```
using ACSSDK.Interfaces;
using ACSSDK.Models;
using AsyscoAMT.Models;
using AsyscoAmt.AsyscoAmt;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using static AsyscoAmt.AsyscoAmt.AsyscoAmtObjects;
using Newtonsoft.Json.Linq;
using static AsyscoAmt.AsyscoAmt.AsyscoAmtEnums;
using static System.Runtime.InteropServices.JavaScript.JSType;
using System.Collections;
using System.Net.NetworkInformation;
using static System.Net.Mime.MediaTypeNames;
using AsyscoAmt;
using System.Text.RegularExpressions;
using Microsoft.VisualBasic;
using AsyscoAMT.AsyscoAmt;

namespace AsyscoAMT.Protocols;

public class TaskProtocol(IIntegration Integration, ILogger Logger) : ITaskProtocol
{
    private readonly ConcurrentDictionary<string, TaskLog> _taskStatuses = new();

    public Task<TaskStatusEvent> CancelTask(ITaskConfig taskConfig)
    {
        var url = Integration.IntegrationInfo.Config.url;
        var existingOpaque = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskConfig.OpaqueConfig);
        var stConfig = JsonConvert.SerializeObject(taskConfig.Config);
        TaskConfig tConfig = JsonConvert.DeserializeObject<TaskConfig>(stConfig);

        var taskId = GetTaskId(taskConfig);
        string token = existingOpaque.Token;
        int batchRequestId = existingOpaque.AmtBatchRequestId.Value;

        Logger.LogInformation($"CancelTask called for task {taskId}");
        var jobStatus = new AsyscoAmtImpl(Logger).KillAmtTask(url, token, batchRequestId, taskId, tConfig).Result;

        return Task.FromResult(new TaskStatusEvent(TaskStatusCode.Canceled, 0, "Killed"));
    }

    public Task<TaskStatusEvent> DeleteTask(ITaskConfig taskConfig)
    {
        return Task.FromResult(new TaskStatusEvent(TaskStatusCode.Canceled, 0, "Deleted"));
    }

    public Task<TaskStatusEvent> StartTask(ITaskConfig taskConfig)
    {

        var stStatus = new TaskStatusEvent(TaskStatusCode.Failed, null, "Failed");

        try
        {
            var existingOpaque = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskConfig.OpaqueConfig ?? "{}") ?? new(0, default, null, 0);
            var stConfig = JsonConvert.SerializeObject(taskConfig.Config);
            TaskConfig taskConfigObj = JsonConvert.DeserializeObject<TaskConfig>(stConfig);

            var url = Integration.IntegrationInfo.Config.url;
            var apiUser = Integration.IntegrationInfo.Config.apiUser;
            var apiUserPassword = Integration.IntegrationInfo.Config.apiUserPassword;

            var application = taskConfigObj.ApplicationName;
 
            IAsyscoAmtLogon? logon = null;
            IAsyscoAmtBatchRequest? batchRequest = null;

            var taskId = Guid.NewGuid();
            HttpClient client = new AsyscoAmtApi(Logger).GetClient(url, null);
            logon = new AsyscoAmtImpl(Logger).PerformAmtLogin(client, url, apiUser, apiUserPassword).Result;
            if (logon is not null)
            {

                var token = logon.AmtLogonToken;
                Logger.LogCritical($"Token Value : {token}");
                client.DefaultRequestHeaders.Add("X-AMT-LogonToken", token);
                //start task
                batchRequest = new AsyscoAmtImpl(Logger).StartAmtTask(client, url, taskId.ToString(), taskConfigObj).Result;
                if (batchRequest is not null)
                {
                    int? batchRequestId = batchRequest.BatchRequestId;
                    if (batchRequestId > 0)
                    {
                        var updateOpaque = existingOpaque with { TaskId = taskId, Token = token, AmtBatchRequestId = batchRequestId };
                        taskConfig.OpaqueConfig = JsonConvert.SerializeObject(updateOpaque);
                        stStatus = new TaskStatusEvent(TaskStatusCode.Running, null, "Running");
                    }
                    else
                    {
                        Logger.LogCritical($"StartTask : Url : Job Start error");
                        var updateOpaque = existingOpaque with { TaskId = taskId, Token = token, AmtBatchRequestId = batchRequestId };
                        taskConfig.OpaqueConfig = JsonConvert.SerializeObject(updateOpaque);
                        stStatus = new TaskStatusEvent(TaskStatusCode.Failed, null, "Failed");
                    }
                }
                else
                {
                    Logger.LogCritical($"StartTask : Url : Job Start error");
                    stStatus = new TaskStatusEvent(TaskStatusCode.Failed, null, "Failed");
                }
            }
            else
            {
                Logger.LogCritical($"StartTask : Url : Authentication error");
                stStatus = new TaskStatusEvent(TaskStatusCode.Failed, null, "Failed");
            }
        }
        catch (Exception ex)
        {
            Logger.LogCritical($"{ex}");
        }
        return Task.FromResult(stStatus);
    }

    public Task<TaskStatusEvent> TaskStatus(ITaskConfig taskConfig)
    {
        var url = Integration.IntegrationInfo.Config.url;
        var existingOpaque = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskConfig.OpaqueConfig);
        var stConfig = JsonConvert.SerializeObject(taskConfig.Config);
        TaskConfig tConfig = JsonConvert.DeserializeObject<TaskConfig>(stConfig);

        TaskStatusEvent returnStatus = new TaskStatusEvent(TaskStatusCode.Failed, null, "Failed");
        var taskId = GetTaskId(taskConfig);
        string token = existingOpaque.Token;
        int batchRequestId = existingOpaque.AmtBatchRequestId.Value;

        Logger.LogInformation($"TaskStatus called for task {taskId}");
        returnStatus = new AsyscoAmtImpl(Logger).StatusAmtTask(url, token, batchRequestId, taskId, tConfig).Result;
        return Task.FromResult(returnStatus);
    }

    private static string GetTaskId(ITaskConfig taskConfig)
    {
        var opaque = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskConfig.OpaqueConfig);
        return opaque.TaskId.ToString();
    }
}

```
It should be noted that the implementation is stateless and each request (CancelTask, DeleteTask, StartTask, TaskStatus) must reference the task opaque config module for
information that can be passed between task requests.

The above code snippet is from the AsyscoAmt TaskProtocol module and shows how the tasks are implemented.
- **CancelTask**       Is used to terminate a task in the remote system. Requires the Agent **Kill** function within OpCon to be enabled. References the TaskOpaqueConfig module for authorization token and batch request id.
- **DeleteTask**       Is called after task completion to perform a cleanup if required.
- **StartTask**        Is used to start a task. Initializes the TaskOpaqueConfig element setting the taskid, the authorization token and the batch request id.
- **TaskStatus**       Is used to determine the status of the task.   References the TaskOpaqueConfig module for taskid, the authorization token and batch request id. When job completion is confirmed, creates the joblog output file.

The task definitions are saved in a TaskConfig object (see Models TaskConfig). The reason for this is to manage the task definitions to determine which values are present and
which values are not present. While it is possible to use the ITaskConfig interface to retrieve definition values, ITaskConfig consists of dynamic values and only contains the
values entered for the task definition. It contains no reference to null values. 

Properties can be used to pass information between task methods and tasks associated with the same schedule. These properties consist of **ScopedProperties** or **Properties**. ScopedProperties equate to schedule instance properties and properties equate to job instance properties. The values are referenced through the ITaskConfig interface which is passed to each task method.

```
    public Task<TaskStatusEvent> StartTask(ITaskConfig taskConfig)
    {
        var existingOpaque = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskConfig.OpaqueConfig ?? "{}") ?? new(0, default);
        var newOpaque = existingOpaque with { TaskId = Guid.NewGuid() };
        taskConfig.OpaqueConfig = JsonConvert.SerializeObject(newOpaque);

        LogConfig("StartTask - Before Config Update", taskConfig);
        taskConfig.Properties["Task Started?"] = "Yes";
        if (!taskConfig.ScopedProperties.TryGetValue("taskStartCount", out var startCount))
        {
            taskConfig.ScopedProperties["taskStartCount"] = "1";
        }
        else
        {
            taskConfig.ScopedProperties["taskStartCount"] = (int.Parse(startCount) + 1).ToString();
        }
        LogConfig("StartTask - After Config Update", taskConfig);

        TaskInstance.StartNew(taskConfig, UpdateTaskStatus, Logger);
        var status = new TaskStatusEvent(TaskStatusCode.NotStarted, null, "NotStarted");
        return Task.FromResult(status);
    }

```
## TaskLogProtocol
The **TaskLogProtocol** must include the **IIntegration Intgration** and **ILogger Logger** interfaces that provides access to the agent information and that writes log messages to the **SMAApiAgentNetcom.log** file in the SAM/Log directory. 

```
using ACSSDK.Interfaces;
using AsyscoAMT.Models;
using AsyscoAmt;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace AsyscoAMT.Protocols;

public class TaskLogProtocol(IIntegration Integration, ILogger Logger) : ITaskLogProtocol
{
    public async Task<string> GetTaskLogFile(IReadOnlyTaskConfig taskInfo, string filePath)
    {
        try
        {
            Logger.LogCritical($"TaskLogProtocol : check for file {filePath}");
            if (File.Exists(filePath))
            {
                var fileContents = await File.ReadAllTextAsync(filePath);
                return fileContents;
            }
            return string.Empty;
        }
        catch
        {
            return string.Empty;
        }
    }

    public Task<IEnumerable<string>> GetTaskLogList(IReadOnlyTaskConfig taskInfo)
    {
        Logger.LogCritical("TaskLogProtocol : get log list");
        var opaqueConfig = JsonConvert.DeserializeObject<TaskOpaqueConfig>(taskInfo.OpaqueConfig);
        var taskId = opaqueConfig.TaskId.ToString();

        var logDir = Path.Join(IntegrationFactory.AssemblyDirectory, "jobOutput");
        var filter = $"AsyscoAmt-{taskId}.log";
        Logger.LogCritical($"TaskLogProtocol : log dir {logDir} filter {filter}");

        var files = Directory.GetFiles(logDir, filter);
        return Task.FromResult<IEnumerable<string>>(files);
     }
}

```
The above code implements two methods, one to retrieve a list of joblogs associated with the task, and the other to retrieve the joblog. The joblog name includes the TaskId of the task created during the StartTask method.