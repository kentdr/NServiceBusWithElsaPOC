# NServiceBus With Elsa POC

A POC to show NServiceBus Elsa activities

## Overview

My goal was to convert the Saga sample in our docs to use dynamic Elsa workflows. The demo sends and receives messages and publishes events through Elsa activities.  The workflows can be built by the designer at runtime.  To run the POC start the ClientUI, Sales, and Billing projects and navigate to the https://localhost:5001/place-order URL (ClientUI project) to send the initial `PlaceOrder` message.  The Sales project should handle the message and publish an `OrderPlaced` event which will be handled by the Billing project.  I removed the Shipping project as I ran out of time before I could look at Sagas.  

## Implementation Details

### `Send a message` and `Publish an event` activities

The code for `Send a message` and `Publish an event` activities can be found [here](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/NServiceBus.Elsa.Activities/SendNServiceBusMessage.cs) and [here](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/NServiceBus.Elsa.Activities/PublishNServiceBusEvent.cs). They both use the `IMessageSession` from NServiceBus to send whatever is passed into the activity as input from the workflow context (see the Elsa documentation on [workflow variables](https://elsa-workflows.github.io/elsa-core/docs/concepts/concepts-workflow-variables) and [workflow context](https://elsa-workflows.github.io/elsa-core/docs/concepts/concepts-workflow-context)).

Both the ClientUI and Sales projects implemented their own Elsa activity for instantiating message and event instances.  ClientUI implemented the [CreatePlaceOrderMessage](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/ClientUI/CreatePlaceOrderMessage.cs) activity, and Sales implemented the [CreateOrderPlacedEvent](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/Sales/CreateOrderPlacedEvent.cs) activity.  The reason for this is I am simply avoiding the quagmire of attempting to create instances of arbitrary types at runtime using a JSON blob and a type string. 

### `When a message is received` activity

The code for the activity can be found [here](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/NServiceBus.Elsa.Activities/NServiceBusMessageReceived.cs).  This activity is a trigger type that signals Elsa to create a new workflow when an event occurs. When a message is received through NServiceBus the workflow is signaled from a [pipeline behavior](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/NServiceBus.Elsa.Activities/CustomElsaHandlerTrigger.cs).  Elsa uses a concept called [Bookmarks](https://elsa-workflows.github.io/elsa-core/docs/next/guides/guides-blocking-activities#bookmarks) to differentiate between multiple running workflows of the same type.  In this case, a [bookmark for the specific message type](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/main/src/NServiceBus.Elsa.Activities/MessageReceivedBookmark.cs) is created when the workflow is initialized.  

## Using the designer

The ClientUI and Sales workflows were created using the designer.  There isn't any code in the projects that send or receive the `PlaceOrder`/`OrderPlaced` messages, it is all done through Elsa.  The workflows can be inspected visually by starting the [ElsaDesigner](https://github.com/kentdr/NServiceBusWithElsaPOC/tree/main/src/ElsaDesigner) project and navigating to the "Workflow Definitions" page.  The code in this project is pulled directly from the Elsa designer tutorial.  

The ClientUI and the Sales endpoints are both configured as Elsa API endpoints.  This means that the designer is able to publish workflows at runtime without the need to recycle the endpoint.  

One downside to the default designer is that it can only point to one of the endpoints at a time. The endpoint to which the designer is pointed is controlled in the `_Host.cshtml` file located [here](https://github.com/kentdr/NServiceBusWithElsaPOC/blob/e444dee5447d59753c7e0fdd5dd2455cec9b47ce/src/ElsaDesigner/Pages/_Host.cshtml#L3).
(the ClientUI project runs on port 5001, and the Sales project runs on port 6001).  

The workflows are saved in a Sqlite database (elsa.sqlite.db) in each of the Client and Sales project directories.  

## Things to consider going forward

While I was successful in proving that events and messages can be sent and received using Elsa activities, there are several things that I was not able to test.  Here is a list of things to consider before any of this could be "production-ready".

### The pipeline behavior will short circuit the pipeline when a workflow is found 
While this may not be a problem, pipeline behavior [execution order is considered non-deterministic within the same stage](https://docs.particular.net/nservicebus/pipeline/manipulate-with-behaviors), so it is conceivable that important downstream behaviors will be skipped when an Elsa workflow is found.  In this implementation, the pipeline had to be short-circuited to avoid NServiceBus throwing an exception due to missing handler classes.  
 
### Transient fault tolerance and message loss
I did not have time to test how Elsa workflows fail or bubble out exceptions.  It is possible a workflow could fail silently and cause NServiceBus to ACK a message that should be retried or failed. If Elsa integration is something we continue pursuing, a concerted effort toward failure and fault tolerance will be an important step.

### The designer is built for a single workflow server by default
In this implementation, each endpoint is its own workflow server which was awkward when using the designer.  If we continue working with Elsa, we will need to come up with a solution to support multiple workflow servers (if there isn't one already) or figure out how the endpoints can collaborate with a single workflow server.

### Dynamic message creation
In this implementation, I opted to make activities for each message type that instantiates them at runtime.  This was very straightforward and simple given the extremely small number of message types (2 in this case).  In a larger-scale implementation, something a bit more dynamic would make things less tedious, though likely more complex to get things right.



