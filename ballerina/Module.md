## Overview

OpenAI is a premier AI research organization focused on developing and promoting friendly AI technologies. The organization aims to create advanced AI models that enhance various applications, ensuring their ethical and responsible use.

The [OpenAI Assistants API](https://platform.openai.com/docs/api-reference/assistants) Connector allows developers to seamlessly integrate OpenAI's advanced language models into their applications. This connector provides tools to build powerful [OpenAI Assistants](https://platform.openai.com/docs/assistants/overview) capable of performing a wide range of tasks, such as generating human-like text, managing conversations with persistent threads, and utilizing multiple tools in parallel. OpenAI has recently announced a variety of new features and improvements to the Assistants API, moving their Beta to a new API version, OpenAI-Beta: assistants=v2. 


## Setup guide

To use the OpenAI Connector, you must have access to the OpenAI API through a [OpenAI Platform account](https://platform.openai.com) and a project under it. If you do not have a OpenAI Platform account, you can sign up for one [here](https://platform.openai.com/signup).

#### Create a OpenAI API Key

1. Open the [OpenAI Platform Dashboard](https://platform.openai.com).


2. Navigate to Dashboard -> API keys
<img src=https://github.com/SanduniU/module-ballerinax-openai-assistants/blob/ecd5fff7d09298d3a491d941875357ac04f970c2/docs/setup/resources/api-key-dashboard.png alt="OpenAI Platform" style="width: 70%;">

3. Click on the "Create new secret key" button
<img src=https://github.com/SanduniU/module-ballerinax-openai-assistants/blob/ecd5fff7d09298d3a491d941875357ac04f970c2/docs/setup/resources/create-new-secret-key.png alt="OpenAI Platform" style="width: 70%;">

4. Fill the details and click on Create secret key
<img src=https://github.com/SanduniU/module-ballerinax-openai-assistants/blob/ecd5fff7d09298d3a491d941875357ac04f970c2/docs/setup/resources/saved-key.png style="width: 70%;">

5. Store the API key securely to use in your application

## Quickstart

A typical integration of the Assistants API has the following flow:

1. **Create an Assistant**
    - Define its custom instructions and pick a model.
    - If helpful, add files and enable tools like Code Interpreter, File Search, and Function calling.

2. **Create a Thread**
    - Create a Thread when a user starts a conversation.

3. **Add Messages to the Thread**
    - Add Messages to the Thread as the user asks questions.

4. **Run the Assistant**
    - Run the Assistant on the Thread to generate a response by calling the model and the tools.

This starter guide walks through the key steps to create and run an Assistant that uses the Code Interpreter tool. In this example, we're creating an Assistant that is a personal math tutor.
### Setting HTTP Headers in Ballerina

Calls to the Assistants API require that you pass a beta HTTP header. This is handled automatically if you’re using OpenAI’s official Python or Node.js SDKs. In Ballerina, you can define the header as follows:

```ballerina
final map<string|string[]> headers = {
    "OpenAI-Beta": ["assistants=v2"]
};
```

### Step 1 : Setting up the connector
To use the `OpenAI Assistants` connector in your Ballerina application, update the `.bal` file as follows:

1. Import the `openai_assistants` module.

```ballerina
import ballerinax/openai_assistants;
```

2. Create a `Config.toml` file and configure the obtained credentials as follows:

```bash
token = "<Access Token>"
```

3. Create a `openai_assistants:Client` with the obtained access token and initialize the connector with it.

```ballerina
configurable string token = ?;

final openai_assistants:Client AssistantClient = check new({
    auth: {
        token
    }
});
```

### Step 2: Create an Assistant

Now, utilize the available connector operations to create an Assistant.



```ballerina
public function main() returns error? {

    // define the required tool
    openai_assistants:AssistantToolsCode tool = {
        type: "code_interpreter"
    };

    // define the assistant request object
    openai_assistants:CreateAssistantRequest request = {
        model: "gpt-3.5-turbo",
        name: "Math Tutor",
        description: "An Assistant for personal math tutoring",
        instructions: "You are a personal math tutor. Help the user with their math questions.",
        tools: [tool]
    };

    // Call the `post assistants` resource to create an Assistant
   var response = check AssistantClient->/assistants.post(request, headers);
    io:println("Assistant ID: ", response);

    if (response is openai_assistants:AssistantObject) {
        io:println("Assistant created: ", response);
    } else {
        io:println("Error: ", response);
    }
}
```

### Step 3: Create a thread

A Thread represents a conversation between a user and one or many Assistants. You can create a Thread when a user (or your AI application) starts a conversation with your Assistant.

```ballerina
    openai_assistants:CreateThreadRequest createThreadReq = {
        messages: []
    };

    // Call the `post threads` resource to create a Thread
    var threadResponse = check AssistantClient->/threads.post(createThreadReq, headers);
    if (threadResponse is openai_assistants:ThreadObject){
        io:println("Thread ID: ", threadResponse.id);
    } else{
        io:println("Error creating thread: ", threadResponse);
    }
```

### Step 4: Add a message to the thread

The contents of the messages your users or applications create are added as Message objects to the Thread. Messages can contain both text and files. There is no limit to the number of Messages you can add to Threads — we smartly truncate any context that does not fit into the model's context window.

```ballerina
    string threadId = "your_thread_id";

    openai_assistants:CreateMessageRequest createMsgReq = {
        role: "user",
        content: "Can you help me solve the equation `3x + 11 = 14`?",
        metadata: {}
    };

    // Create a message in the thread
    var messageResponse = check AssistantClient->/threads/[threadId]/messages.post(createMsgReq, headers);
    if (messageResponse is openai_assistants:MessageObject){
        io:println("Created Message: ", messageResponse);
    } else {
        io:println("Error creating Message: ", messageResponse);
    }
```


### Step 5: Create a run

Once all the user Messages have been added to the Thread, you can Run the Thread with any Assistant. Creating a Run uses the model and tools associated with the Assistant to generate a response. These responses are added to the Thread as assistant Messages.

```ballerina
   string threadId = "your_thread_id";

    openai_assistants:CreateRunRequest runReq = {
        assistant_id: "your_assistant_id",
        model: "gpt-3.5-turbo",
        instructions: "You are a personal math tutor. Assist the user with their math questions.",
        temperature: 0.7,
        top_p: 0.9,
        max_prompt_tokens: 400,
        max_completion_tokens: 200
    };

    // Create a run in the thread
    var resp = AssistantClient->/threads/[threadId]/runs.post(runReq, headers);
    if (resp is openai_assistants:RunObject) {
        io:println("Created Run: ", resp);
    } else {
        io:println("Error creating run: ", resp);
    }
```
Once the Run completes, you can list the Messages added to the Thread by the Assistant.
```
    string threadId = "your_thread_id";

    map<string|string[]> headers = {
        "OpenAI-Beta": ["assistants=v2"]
    };

    // List messages in the thread
    var res = AssistantClient->/threads/[threadId]/messages.get(headers);
    if (res is openai_assistants:ListMessagesResponse) {
        io:println("Messages of Thread: ", res);
    } else {
        io:println("Error retrieving messages: ", res);
    }

```

## Examples

The `OpenAI Assistants` connector provides practical examples illustrating usage in various scenarios. Explore these [examples](https://github.com/module-ballerinax-openai-assistants/tree/main/examples/), covering the following use cases:

[//]: # (TODO: Add examples)
