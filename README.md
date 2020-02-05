# CSVBlobToEventHub
Function to copy data from CSV and submit via function to Event Hubs

This project is a function app that collects CSVs from Blob and injects the rows one at a time into an Event Hub instance with optional delay. This is written in C# but you could achieve the same in other languages.

## Introduction

There are several pieces involved in writing a function app using triggers, inputs and outputs. You'll need to know about the following:

* Triggers
* Input Bindings
* Output Bindings
* Async apps

### Triggers and bindings

A trigger will be what causes your Function to run. An Input is optional, and used to collect data if your trigger does not provide it. Outputs are where your data is sent by the function. A full list of bindings can be found at [functions-triggers-bindings](https://docs.microsoft.com/en-us/azure/azure-functions/functions-triggers-bindings) and each one links to a full page of documentation explaining how to use that binding.

### Async Apps

Some triggers will start an app and have no data, for instance a timer. Others will start an app with one piece of data. In this scenario, the app will do something with the data and then when the app is finished, the data is returned to the output binding and the app terminates. Finally, you may have a trigger which provides multiple data in an array. In this instance you cannot simply return the data and terminate the app since you need to process all of the data. Here you should configure the app as async and return the data piece by piece until you run out and only then terminate the app. Each trigger, input and output will detail the ways of using it, but you may need to look carefully to notice the async keyword.

This example [from the Event Hub documentation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs?tabs=csharp#output) shows the return method if you have one input:

```CSHARP
[FunctionName("EventHubOutput")]
[return: EventHub("outputEventHubMessage", Connection = "EventHubConnectionAppSetting")]
public static string Run([TimerTrigger("0 */5 * * * *")] TimerInfo myTimer, ILogger log)
{
    log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");
    return $"{DateTime.Now}";
}
```

And this example [from the Event Hub documentation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs?tabs=csharp#output) shows the async method where you use an IASyncCollector object to iterate through data:

```CSHARP
[FunctionName("EH2EH")]
public static async Task Run(
    [EventHubTrigger("source", Connection = "EventHubConnectionAppSetting")] EventData[] events,
    [EventHub("dest", Connection = "EventHubConnectionAppSetting")]IAsyncCollector<string> outputEvents,
    ILogger log)
{
    foreach (EventData eventData in events)
    {
        // do some processing:
        var myProcessedEvent = DoSomething(eventData);

        // then send the message
        await outputEvents.AddAsync(JsonConvert.SerializeObject(myProcessedEvent));
    }
}
```

Note the "public static async Task" which will also need [System.Threading.Tasks](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks?view=netframework-4.8) to be added to the code to support the methods used.

## Setup

This demo will require the following components:

* Function App
* Storage Account (Blob Containers)
* Event Hub

Optionally, you may also want to use Stream Analytics and Power BI to visualise the data, but I won't go into those here. You'll be able to see data hitting the Event Hub and can use [Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer) to view the messages on the bus.