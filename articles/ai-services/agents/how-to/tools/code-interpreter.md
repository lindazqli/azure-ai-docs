---
title: 'How to use Azure AI Agent service Code Interpreter'
titleSuffix: Azure OpenAI
description: Learn how to use Azure AI Agent service Code Interpreter
services: cognitive-services
manager: nitinme
ms.service: azure
ms.topic: how-to
ms.date: 12/11/2024
author: aahill
ms.author: aahi
ms.custom: azure-ai-agents
zone_pivot_groups: selection-code-interpreter
---

# Azure AI Agent service Code Interpreter



::: zone pivot="overview"

Code Interpreter allows the agents to write and run Python code in a sandboxed execution environment. With Code Interpreter enabled, your agent can run code iteratively to solve more challenging code, math, and data analysis problems. When your Agent writes code that fails to run, it can iterate on this code by modifying and running different code until the code execution succeeds.

> [!IMPORTANT]
> Code Interpreter has [additional charges](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/) beyond the token based fees for Azure OpenAI usage. If your Agent calls Code Interpreter simultaneously in two different threads, two code interpreter sessions are created. Each session is active by default for one hour.

### Supported models

The [models page](../../quotas-limits.md) contains the most up-to-date information on regions/models where agents and code interpreter are supported.

We recommend using Agents with the latest models to take advantage of the new features, larger context windows, and more up-to-date training data.

::: zone-end

::: zone pivot="code-example"


## Define imports and create a project client 

# [Python](#tab/python)

To use code interpreter, first add the `import` statements shown in the example, and create a project client, which will contain a connection string to your AI project, and will be used to authenticate API calls. 

```python
import os
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import CodeInterpreterTool
from azure.ai.projects.models import FilePurpose
from azure.identity import DefaultAzureCredential
from pathlib import Path

# Create an Azure AI Client from a connection string, copied from your AI Foundry project.
# At the moment, it should be in the format "<HostName>;<AzureSubscriptionId>;<ResourceGroup>;<HubName>"
# Customer needs to login to Azure subscription via Azure CLI and set the environment variables
project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(), conn_str=os.environ["PROJECT_CONNECTION_STRING"]
)

```

# [C#](#tab/csharp)

To use code interpreter, first you need to create a project client, which will contain a connection string to your AI project, and will be used to authenticate API calls.

```csharp
var connectionString = Environment.GetEnvironmentVariable("PROJECT_CONNECTION_STRING");
AgentsClient client = new AgentsClient(connectionString, new DefaultAzureCredential()); 
```

---

## Upload a file 

# [Python](#tab/python)

Upload the file using the `upload_and_poll()` function, specifying the file path and the `FilePurpose.AGENTS` purpose.

```python
# Upload a file and add it to the client 
file = project_client.agents.upload_file_and_poll(
    file_path="nifty_500_quarterly_results.csv", purpose=FilePurpose.AGENTS
)
print(f"Uploaded file, file ID: {file.id}")

```
# [C#](#tab/csharp)

Files can be uploaded and then referenced by agents or messages. First, use the generalized upload API with a `purpose` of `Agents` to make a file ID available. Once uploaded, the file ID can then be provided to create a vector store for it. The vector store ID can then be provided to an agent upon creation. 

> [!NOTE]
> You do not need to provide `toolResources` if you don't create a vector store.

```csharp
// Upload a file and wait for it to be processed
Response<AgentFile> uploadAgentFileResponse = await client.UploadFileAsync(
    filePath: "sample_file_for_upload.txt",
    purpose: AgentFilePurpose.Agents);

AgentFile uploadedAgentFile = uploadAgentFileResponse.Value;

// Create a vector store with the file and wait for it to be processed.
// If you do not specify a vector store, create_message will create a vector store with a default expiration policy of seven days after they were last active
VectorStore vectorStore = await client.CreateVectorStoreAsync(
    fileIds:  new List<string> { uploadedAgentFile.Id },
    name: "my_vector_store");

CodeInterpreterToolResource codeInterpreterToolResource = new CodeInterpreterToolResource();
CodeInterpreterToolResource.VectorStoreIds.Add(vectorStore.Id);
```
---

## Create an agent with the code interpreter tool

# [python](#tab/python)

Define the `code_interpreter` tool with `CodeInterpreterTool()` and include the file ID of the file you uploaded. Afterwards, create the agent with `tools` set to `code_interpreter.definitions` and `tool_resources` set to `code_interpreter.resources`.

```python

code_interpreter = CodeInterpreterTool(file_ids=[file.id])

# create agent with code interpreter tool and tools_resources
agent = project_client.agents.create_agent(
    model="gpt-4o-mini",
    name="my-agent",
    instructions="You are helpful agent",
    tools=code_interpreter.definitions,
    tool_resources=code_interpreter.resources,
)
```

# [C#](#tab/csharp)

```csharp
Response<Agent> agentResponse = await client.CreateAgentAsync(
    model: "gpt-4o-mini",
    name: "My agent",
    instructions: "You are a helpful agent.",
    tools: new List<ToolDefinition> { new CodeInterpreterToolDefinition() },
    toolResources: new ToolResources() { CodeInterpreter = codeInterpreterToolResource });
Agent agent = agentResponse.Value;
```

---

## Create a thread, message, and get the agent response 

# [Python](#tab/python)

Next create a thread with `create_thread()` and attach a message to it using `create_message()` that will trigger the code interpreter tool. Afterwards, create and execute a run with `create_and_process_run()`. Once the run finishes, you can delete the file from the agent with `delete_file()` to free up space in the agent. Finally, print the messages from the agent.

```python
# create a thread
thread = project_client.agents.create_thread()
print(f"Created thread, thread ID: {thread.id}")

# create a message
message = project_client.agents.create_message(
    thread_id=thread.id,
    role="user",
    content="Could you please create bar chart in the TRANSPORTATION sector for the operating profit from the uploaded csv file and provide file to me?",
)
print(f"Created message, message ID: {message.id}")

# create and execute a run
run = project_client.agents.create_and_process_run(thread_id=thread.id, assistant_id=agent.id)
print(f"Run finished with status: {run.status}")

if run.status == "failed":
    # Check if you got "Rate limit is exceeded.", then you want to get more quota
    print(f"Run failed: {run.last_error}")

# delete the original file from the agent to free up space (note: this does not delete your version of the file)
project_client.agents.delete_file(file.id)
print("Deleted file")

# print the messages from the agent
messages = project_client.agents.get_messages(thread_id=thread.id)
print(f"Messages: {messages}")

# get the most recent message from the assistant
last_msg = messages.get_last_text_message_by_sender("assistant")
if last_msg:
    print(f"Last Message: {last_msg.text.value}")
```

# [C#](#tab/csharp)

Next create a thread with `CreateThreadAsync()` and a thread with `CreateMessageAsync()`. After the thread is created, you can add messages to it with `CreateMessageAsync()` that will cause the code interpreter tool to trigger. Create a run, and then continue polling it until it reaches a terminal status. Assuming the run was successful, parse the agent's response by listing the messages.

```csharp
//Create a thread
Response<AgentThread> threadResponse = await client.CreateThreadAsync();
AgentThread thread = threadResponse.Value;

//With a thread created, messages can be created on it:
Response<ThreadMessage> messageResponse = await client.CreateMessageAsync(
    thread.Id,
    MessageRole.User,
    "I need to solve the equation `3x + 11 = 14`. Can you help me?");
ThreadMessage message = messageResponse.Value;

//A run can then be started that evaluates the thread against an agent:
Response<ThreadRun> runResponse = await client.CreateRunAsync(
    thread.Id,
    agent.Id,
    additionalInstructions: "Please address the user as Jane Doe. The user has a premium account.");
ThreadRun run = runResponse.Value;

//Once the run has started, it should then be polled until it reaches a terminal status:
do
{
    await Task.Delay(TimeSpan.FromMilliseconds(500));
    runResponse = await client.GetRunAsync(thread.Id, runResponse.Value.Id);
}
while (runResponse.Value.Status == RunStatus.Queued
    || runResponse.Value.Status == RunStatus.InProgress);

//Assuming the run successfully completed, listing messages from the thread that was run will now reflect new information added by the agent:

Response<PageableList<ThreadMessage>> afterRunMessagesResponse
    = await client.GetMessagesAsync(thread.Id);
IReadOnlyList<ThreadMessage> messages = afterRunMessagesResponse.Value.Data;

// Note: messages iterate from newest to oldest, with the messages[0] being the most recent
foreach (ThreadMessage threadMessage in messages)
{
    Console.Write($"{threadMessage.CreatedAt:yyyy-MM-dd HH:mm:ss} - {threadMessage.Role,10}: ");
    foreach (MessageContent contentItem in threadMessage.ContentItems)
    {
        if (contentItem is MessageTextContent textItem)
        {
            Console.Write(textItem.Text);
        }
        else if (contentItem is MessageImageFileContent imageFileItem)
        {
            Console.Write($"<image from ID: {imageFileItem.FileId}");
        }
        Console.WriteLine();
    }
}
```
---

## Download files generated by code interpreter

# [Python](#tab/python)

Files generated by Code Interpreter can be found in the Agent message responses. You can download image file generated by code interpreter, by iterating through the response's `image_contents` and calling `save_file()` with a name and the file ID.  

```python
# save the newly created file
for image_content in messages.image_contents:
  print(f"Image File ID: {image_content.image_file.file_id}")
  file_name = f"{image_content.image_file.file_id}_image_file.png"
  project_client.agents.save_file(file_id=image_content.image_file.file_id, file_name=file_name)
  print(f"Saved image file to: {Path.cwd() / file_name}") 
```

# [C#](#tab/csharp)

Files generated by code interpreter can be found in the Agent message responses. You can download image file generated by code interpreter by iterating through the response's messages and checking for an ImageFileId. If that field exists, use the following code:

```csharp
foreach (MessageContent contentItem in message.Content)
{
    if (!string.IsNullOrEmpty(contentItem.ImageFileId))
    {
        OpenAIFileInfo imageInfo = await fileClient.GetFileAsync(contentItem.ImageFileId);
        BinaryData imageBytes = await fileClient.DownloadFileAsync(contentItem.ImageFileId);
        using FileStream stream = File.OpenWrite($"{imageInfo.Filename}.png");
        imageBytes.ToStream().CopyTo(stream);

        Console.WriteLine($"<image: {imageInfo.Filename}.png>");
    }
}
```

---

::: zone-end

::: zone pivot="supported-filetypes"


### Supported file types

|File format|MIME Type|
|---|---|
|`.c`| `text/x-c` |
|`.cpp`|`text/x-c++` |
|`.csv`|`application/csv`|
|`.docx`|`application/vnd.openxmlformats-officedocument.wordprocessingml.document`|
|`.html`|`text/html`|
|`.java`|`text/x-java`|
|`.json`|`application/json`|
|`.md`|`text/markdown`|
|`.pdf`|`application/pdf`|
|`.php`|`text/x-php`|
|`.pptx`|`application/vnd.openxmlformats-officedocument.presentationml.presentation`|
|`.py`|`text/x-python`|
|`.py`|`text/x-script.python`|
|`.rb`|`text/x-ruby`|
|`.tex`|`text/x-tex`|
|`.txt`|`text/plain`|
|`.css`|`text/css`|
|`.jpeg`|`image/jpeg`|
|`.jpg`|`image/jpeg`|
|`.js`|`text/javascript`|
|`.gif`|`image/gif`|
|`.png`|`image/png`|
|`.tar`|`application/x-tar`|
|`.ts`|`application/typescript`|
|`.xlsx`|`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`|
|`.xml`|`application/xml` or `text/xml`|
|`.zip`|`application/zip`|

::: zone-end


## See also

* Learn more [about agents](../../overview.md).