# Post to Cosmos DB from Azure function passing parameters from request body only

_In this post I am going to explain how to do a post to [Azure Cosmos Db](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction). The tutorials I came across with all started from the auto-generated code in the [azure functions](https://azure.microsoft.com/en-us/services/functions/) portal, and would allow to pass in data through query parameters. I do not really like to pass data through query params because I want the route to look like HTTP POST /message. Here I will show my working example to post a document to Cosmos Db using only data from the request body._

I expect you already have some basic knowledge of Azure Functions and Cosmos Db, so I will not include the steps for adding a [Cosmos DB output binding](https://t.co/xorlB0uIhF) here. I will only elaborate on the `function.json` file and the `run.csx` file.


My function.json:
```
{
  "bindings": [
    {
      "authLevel": "function",
      "name": "req",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "post"
      ],
      "route": "message"
    },
    {
      "type": "documentDB",
      "name": "$return",
      "databaseName": "XXX",
      "collectionName": "messageCollection",
      "createIfNotExists": true,
      "connection": "XXX_DOCUMENTDB",
      "direction": "out"
    }
  ],
  "disabled": false
}
```
You only need 2 bindings for your function. The `function.json` file contains a input HTTP binding and a output binding to Cosmos DB. When you create the function in the portal, it will autogenerate a output HTTP binding also. That has to be deleted. 

The HTTP input binding is quite straight forward. It specifies that it only listens to a HTTP Post request on the `message` route. This makes it possible to have a HTTP Get request on the same route for example.

When you add an output binding for Cosmos Db, make sure that the `name` of the binding is set to `$return`. The alternative would be to specify an `out parameter` in the method definition, but that is not supported for `async` functions. We need an `async` function because we are reading the request body asynchronously. The output binding to Cosmos Db will take care of writing the return variable from the function to the `messageCollection`.

my run.csx:
```
using System.Net;

public async static Task<object> Run(HttpRequestMessage req, TraceWriter log)
{
    dynamic data = await req.Content.ReadAsAsync<object>();

    log.Info($"Incoming request body = {data}");

    var id = data?.id;
    var message = data?.message;
    var duedate = data?.duedate;

    var messageDocument = new {
        id = id,
        duedate = duedate.ToString(),
        message = message,
        timestamp = DateTime.UtcNow
    };

    if (id != "" && message != "") {
        return messageDocument;
    }
    else
    {
        return null;
    }
}
```
In our function code, apart from the logger, we only expect the `request` object as a parameter. We read from the request body with this line of code: 

```
dynamic data = await req.Content.ReadAsAsync<object>();
```

This code will create an object to return from our function:

```
var id = data?.id;
var message = data?.message;
var duedate = data?.duedate;

var messageDocument = new {
    id = id,
    duedate = duedate.ToString(),
    message = message,
    timestamp = DateTime.UtcNow
};
```

Mind thata we did not specify a HTTP output binding, but the function will automatically return the correct HTTP status code to the caller. 

That is my solution. I had some problems overcoming the `async` function not allowin a `output parameter` for the output binding to Cosmos Db, but I actually like my solutions to remove the HTTP output binding, since the function returns a valid HTTP reponse anyway. 

If you have any suggestions for a better solution, or just another approach to get to the same result, pls let me know in the comments!
