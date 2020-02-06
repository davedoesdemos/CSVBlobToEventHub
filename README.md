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

This example [from the Event Hub documentation](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs?tabs=csharp#output) shows the return method if you have one output:

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

## The Code

### host.json

This file simply lists the version and sets up the Event Hub extension to be used by the app as per the documentation. It's pretty self explanatory but is necessary to be added or the extension is not recognised.

```json
{
  "version": "2.0",
    "extensions": {
      "eventHubs": {
        "batchCheckpointFrequency": 5,
        "eventProcessorOptions": {
          "maxBatchSize": 256,
          "prefetchCount": 512
        }
      }
    }
}
```

### Function1.cs

At the top of the function, we set up the trigger, in this case BlobTrigger, using a connection with the name csvblobstore. This connection is configured either in your local settings if testing locally, or in the Function App settings in Azure. We set this up as a Stream variable named myBlob which means we can just treat it as a file in C#. We also set up the Event Hub output as an IAsyncCollector which we can write output events to. The whole function is async here as we will be sending multiple outputs from one input.

```CSHARP
public static class Function1
    {
        [FunctionName("Function1")]
        public static async Task Run([BlobTrigger("csvblobs/{name}", Connection = "csvblobstore")]Stream myBlob, [EventHub("dest", Connection = "eventhubconnection")]IAsyncCollector<string> outputEvents, string name, ILogger log)
```

Next, we write to the log some useful information. This is straight from sample code and not necessary but is recommended for troubleshooting. If you're wanting to tweak for performance you may want to leave this out if you're very confident in the app code and happy to use other methods for troubleshooting. We then set up a StreamReader to read the Blob line by line to process the CSV file. Please note that I included no code at all to check that we have a valid CSV file. This demo is all about setting up a function app so I left out everything but that.

```CSHARP
        {
            log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
            using (var reader = new StreamReader(myBlob))
            {
```
Next, we read the first line and assume it's a header. If it wasn't, the JSON would use the first row as headers regardless and end up with some weird attribute names. Again, this is a minimum viable product to show Function Apps. We split the line by comma and create an array.

```CSHARP
                //Read the first line of the CSV file and break into header values
                var line = reader.ReadLine();
                log.LogInformation("headers: " + line);
                var headers = line.Split(',');
```
Because it's useful to have a stream of events last for a while I added a delay, defined here. If you don't do this Event Hubs will just suck up all of the rows immediately (it's very capable!) so you won't get ongoing events for testing of other technologies. I made this app for testing events in Stream Analytics or PowerBI or some other Event Hub driven thing, maybe CosmosDB too.

```CSHARP
                //Define the sleep interval in milliseconds, use 0 if you don't want to wait, or remove the sleep command below
                int delayTime = 1000;
```
Next, we do a while loop to read the remainder of the file line by line and make another array of the values. Since we want the rows to be sent as a JSON one at a time to create many events from one file, we set up an empty string. There are certainly ways to use Newtonsoft functionality to create JSON from an object, but since CSV is 2D data it's easy enough to manually create it. For each line, we add the current time to simulate a transaction time and then we also add the columns and values one at a time in a for loop. The final value can't have a comma in JSON so we add it outside the loop without the comma.

```CSHARP
                //Read the rest of the file
                while (!reader.EndOfStream)
                {
                    //Create an empty string for our JSON
                    string outputJSON = "";
                    outputJSON = outputJSON + "{\n";
                    
                    //Read our lines one by one and split into values
                    line = reader.ReadLine();
                    var values = line.Split(',');
                    log.LogInformation("Values: " + line);
                    
                    //Add a datestamp to the data
                    outputJSON = outputJSON + "  functionDateStamp: \"" + DateTime.Now.ToString() + "\",\n";

                    //Add all of the data except the last value
                    for (int i = 0; i < (values.Length - 1); i++)
                    {
                        outputJSON = outputJSON + "  \"" + headers[i] + "\": \"" + values[i] + "\",\n";
                    }
                    //Add the last value without a comma to properly form the JSON
                    int j = values.Length - 1;
                    outputJSON = outputJSON + "  \"" + headers[j] + "\": \"" + values[j] + "\"\n";
                    //Close the JSON
                    outputJSON = outputJSON + "}";
```
Finally we add outputJSON, the string we created to the outputEvents variable, which is our output binding to Event Hubs. We do this with an await command to spawn a new thread task to avoid delaying the next line being processed. The main thread then carries on and reads the next line without delay. This makes the app extremely performant.

```CSHARP
                    await outputEvents.AddAsync(outputJSON);
                    log.LogInformation("Added: " + outputJSON);
```
We then sleep the main thread (optionally) to spread the events out. Don't put this in a real Function as it does nothing useful in the real world. Event Hubs will easily accept your events at whatever speed you supply them!

```CSHARP
                    //Sleep the function to delay the next event (otherwise it's really fast!)
                    System.Threading.Thread.Sleep(delayTime);
                }
            }
        }
    }
```