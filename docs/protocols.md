---
slug: '/'
sidebar_label: 'Protocols'
---

# Developing ACS Protocols

The protocol classes implement the link between the ACS environment and the remote system. Separate protocol classes must be created for managing the communications link, submitting and managing tasks and retrieving task logs. 

Create a folder **Protocols**.
In this folder create the **IntegrationProtocol**, **TaskLogProtocol** and **TaskProtocol** classes.

The IntegrationProtocol class manages the connection between the ACS implementation and the remote system. After configuration has been completed and the ACS implementation is marked-up, the **InitialStatus** method is called to establish the connection to the remote system. If the connection is successfully established, a **Communicating** status is returned or if unsuccessful a **NotCommunicating** status is returned. If a **Communicating** status is returned, it will be possible to submit task requests to the remote system.

Once the connection is established, the **Status** method is called on a regular basis to determine if the remote system is still available top process task requests. If available a **Communicating** status is returned, if not available a **NotCommunicating** status is returned and the ACS implementation will be marked-down.    

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

    public async Task<Result> InitialStatus(IConfig integrationConfig)
    {
        var url = integrationConfig?.Config?.apiUrl.ToString();
        var user = integrationConfig?.Config?.apiUser.ToString();
        var password = integrationConfig?.Config?.apiUserPassword.ToString();
        // var opaqueConfig = JsonConvert.DeserializeObject<IntegrationOpaqueConfig>(integrationConfig.OpaqueConfig ?? "{}") ?? new(null);

        if (string.IsNullOrEmpty(url) || string.IsNullOrEmpty(user) || string.IsNullOrEmpty(password))
        {
            Logger.LogCritical("Communication can not be established : Invalid Configuration");
            return new Result(ResultCode.NotCommunicating);
        }

        // TODO implement connection to remote system
        // success 
        return new Result(ResultCode.Communicating);
        // failure 
        return new Result(ResultCode.NotCommunicating);
    }

    public Task<Result> Status(IConfig integrationConfig)
    {
        var url = integrationConfig?.Config?.apiUrl.ToString();
        var user = integrationConfig?.Config?.apiUser.ToString();
        var password = integrationConfig?.Config?.apiUserPassword.ToString();

        // TODO implement connection check to remote system
        // success 
        return new Result(ResultCode.Communicating);
        // failure 
        return new Result(ResultCode.NotCommunicating);

    }

}

```
The **IntegrationProtocol** class must implement the **IIntegrationProtocol** interface. The class constructor should include the **ILogger Logger** interface that writes log messages to the **SMAApiAgentNetcom.log** file in the SAM/Log directory. 

The class must contain the **InitialStatus** and **Status** methods. In each case the methods accept the **IConfig** interface which provides the ACS agent configuration information.
When referencing the configuration information, the actual name value in the line pf code integrationConfig?Config.**apiUrl**.ToString() is the name of the resource defined in the **IntegrationSchemaGeneration** class definition.

