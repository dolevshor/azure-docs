---
title: "Azure SignalR Service serverless quickstart - C#"
description: "A quickstart for using Azure SignalR Service and Azure Functions to create an app showing GitHub star count using C#."
author: vicancy
ms.service: signalr
ms.devlang: csharp
ms.topic: quickstart
ms.custom: devx-track-csharp, mode-other
ms.date: 03/30/2022
ms.author: lianwei
---

# Quickstart: Create an app showing GitHub star count with Azure Functions and SignalR Service via C#

In this article, you'll learn how to use SignalR Service and Azure Functions to build a serverless application with C# to broadcast messages to clients.

> [!NOTE]
> You can get the code mentioned in this article from [GitHub](https://github.com/aspnet/AzureSignalR-samples/tree/main/samples/QuickStartServerless/csharp).

## Prerequisites

The following prerequisites are needed for this quickstart:

- Visual Studio Code, or other code editor. If you don't already have Visual Studio Code installed, [download Visual Studio Code here](https://code.visualstudio.com/Download).
- An Azure subscription. If you don't have an Azure subscription, [create one for free](https://azure.microsoft.com/free/dotnet) before you begin.
- [Azure Functions Core Tools](../azure-functions/functions-run-local.md?tabs=windows%2Ccsharp%2Cbash#v2)
- [.NET Core SDK](https://dotnet.microsoft.com/download)

## Create an Azure SignalR Service instance

[!INCLUDE [Create instance](includes/signalr-quickstart-create-instance.md)]

## Setup and run the Azure Function locally

You'll need the Azure Functions Core Tools for this step.

1. Create an empty directory and change to the directory with the command line.
1. Initialize a new project.

    ```bash
    # Initialize a function project
    func init --worker-runtime dotnet

    # Add SignalR Service package reference to the project
    dotnet add package Microsoft.Azure.WebJobs.Extensions.SignalRService
    ```

1. Using your code editor, create a new file with the name *Function.cs*. Add the following code to *Function.cs*:

    ```csharp
    using System;
    using System.IO;
    using System.Linq;
    using System.Net.Http;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Http;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.Azure.WebJobs.Extensions.SignalRService;
    using Newtonsoft.Json;
    
    namespace CSharp
    {
        public static class Function
        {
            private static HttpClient httpClient = new HttpClient();
            private static string Etag = string.Empty;
            private static string StarCount = "0";
    
            [FunctionName("index")]
            public static IActionResult GetHomePage([HttpTrigger(AuthorizationLevel.Anonymous)]HttpRequest req, ExecutionContext context)
            {
                var path = Path.Combine(context.FunctionAppDirectory, "content", "index.html");
                return new ContentResult
                {
                    Content = File.ReadAllText(path),
                    ContentType = "text/html",
                };
            }
    
            [FunctionName("negotiate")]
            public static SignalRConnectionInfo Negotiate( 
                [HttpTrigger(AuthorizationLevel.Anonymous)] HttpRequest req,
                [SignalRConnectionInfo(HubName = "serverless")] SignalRConnectionInfo connectionInfo)
            {
                return connectionInfo;
            }
    
            [FunctionName("broadcast")]
            public static async Task Broadcast([TimerTrigger("*/5 * * * * *")] TimerInfo myTimer,
            [SignalR(HubName = "serverless")] IAsyncCollector<SignalRMessage> signalRMessages)
            {
                var request = new HttpRequestMessage(HttpMethod.Get, "https://api.github.com/repos/azure/azure-signalr");
                request.Headers.UserAgent.ParseAdd("Serverless");
                request.Headers.Add("If-None-Match", Etag);
                var response = await httpClient.SendAsync(request);
                if (response.Headers.Contains("Etag"))
                {
                    Etag = response.Headers.GetValues("Etag").First();
                }
                if (response.StatusCode == System.Net.HttpStatusCode.OK)
                {
                    var result = JsonConvert.DeserializeObject<GitResult>(await response.Content.ReadAsStringAsync());
                    StarCount = result.StarCount;
                }
                
                await signalRMessages.AddAsync(
                    new SignalRMessage
                    {
                        Target = "newMessage",
                        Arguments = new[] { $"Current star count of https://github.com/Azure/azure-signalr is: {StarCount}" }
                    });
            }
    
            private class GitResult
            {
                [JsonRequired]
                [JsonProperty("stargazers_count")]
                public string StarCount { get; set; }
            }
        }
    }
    ```

    The code in *Function.cs* has three functions:
    - `GetHomePage` is used to get a website as client.
    - `Negotiate` is used by the client to get an access token.
    - `Broadcast` is periodically called to get the star count from GitHub and then broadcast messages to all clients.

1. The client interface for this sample is a web page. We render the web page using the `GetHomePage` function by reading HTML content from file *content/index.html*. Now let's create this *index.html* under the `content` subdirectory with the following content:

    ```html
    <html>

    <body>
      <h1>Azure SignalR Serverless Sample</h1>
      <div id="messages"></div>
      <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/3.1.7/signalr.min.js"></script>
      <script>
        let messages = document.querySelector('#messages');
        const apiBaseUrl = window.location.origin;
        const connection = new signalR.HubConnectionBuilder()
            .withUrl(apiBaseUrl + '/api')
            .configureLogging(signalR.LogLevel.Information)
            .build();
          connection.on('newMessage', (message) => {
            document.getElementById("messages").innerHTML = message;
          });

          connection.start()
            .catch(console.error);
      </script>
    </body>

    </html>
    ```

1. Update your `*.csproj` to make the content page in the build output folder.

    ```html
    <ItemGroup>
      <None Update="content/index.html">
        <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
      </None>
    </ItemGroup>
    ```

1. It's almost done now. The last step is to set a connection string of the SignalR Service to Azure Function settings.

    1. Confirm the SignalR Service instance was successfully created by searching for its name in the search box at the top of the portal. Select the instance to open it.

        ![Search for the SignalR Service instance](media/signalr-quickstart-azure-functions-csharp/signalr-quickstart-search-instance.png)

    1. Select **Keys** to view the connection strings for the SignalR Service instance.

        ![Screenshot that highlights the primary connection string.](media/signalr-quickstart-azure-functions-javascript/signalr-quickstart-keys.png)

    1. Copy the primary connection string, and then run the following command:

        ```bash
        func settings add AzureSignalRConnectionString "<signalr-connection-string>"
        ```

1. Run the Azure function locally:

    ```bash
    func start
    ```

    After the Azure function is running locally, open `http://localhost:7071/api/index` and you can see the current star count. If you star or unstar in the GitHub, you'll get a star count refreshing every few seconds.

    > [!NOTE]
    > SignalR binding needs Azure Storage, but you can use a local storage emulator when the function is running locally.
    > If you got the error `There was an error performing a read operation on the Blob Storage Secret Repository. Please ensure the 'AzureWebJobsStorage' connection string is valid.` You need to download and enable [Storage Emulator](../storage/common/storage-use-emulator.md)

[!INCLUDE [Cleanup](includes/signalr-quickstart-cleanup.md)]

Having issues? Try the [troubleshooting guide](signalr-howto-troubleshoot-guide.md) or [let us know](https://aka.ms/asrs/qspython).

## Next steps

In this quickstart, you built and ran a real-time serverless application locally. Next, learn more about bi-directional communication between clients and Azure Functions with Azure SignalR Service.

> [!div class="nextstepaction"]
> [SignalR Service bindings for Azure Functions](../azure-functions/functions-bindings-signalr-service.md)

> [!div class="nextstepaction"]
> [Azure Functions Bi-directional communicating sample](https://github.com/aspnet/AzureSignalR-samples/tree/main/samples/BidirectionChat)

> [!div class="nextstepaction"]
> [Azure Functions Bi-directional communicating sample for isolated process](https://github.com/aspnet/AzureSignalR-samples/tree/main/samples/DotnetIsolated-BidirectionChat)

> [!div class="nextstepaction"]
> [Deploy to Azure Function App using Visual Studio](../azure-functions/functions-develop-vs.md#publish-to-azure)
