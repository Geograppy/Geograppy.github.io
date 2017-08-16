# Query Cosmos db from Azure fuction with HTTP Trigger using POCO input parameter

_I had some trouble to implement a simple HTTP GET method in azure functions. It was quite a struggle so let me explain how I got it to work eventually. I wanted the route to look like `HTTP GET /message/{messageid}`. My code compiled just fine, and Log.Info statements worked also. The query result was `null` though. The function returns HTTP 404 which seems right in this case._

You can find some usefule resources for  [Azure Functions Cosmos DB bindings](https://t.co/xorlB0uIhF) and [Store unstructured data using Azure Functions and Cosmos DB](https://t.co/1KpbJIBMFx), but they do not really explain how the parameters from the HTTP trigger get passed into the Cosmos db input binding to query your documentCollection. This resource  on [stackoverflow](https://stackoverflow.com/questions/42829465/accessing-documentdb-from-azure-functions-with-dynamic-documentid) set me on the right track, but still it took me some time to get all the bits and pieces right. 

Here is my working example:

My function.json:

``` 
{
  "bindings": [
    {
      "authLevel": "function",
      "name": "input",
      "type": "httpTrigger",
      "direction": "in",
      "methods": [
        "get"
      ]
    },
    {
      "name": "$return",
      "type": "http",
      "direction": "out"
    },
    {
      "type": "documentDB",
      "name": "messageDocument",
      "databaseName": "XXX",
      "collectionName": "messageCollection",
      "id": "{messageid}",
      "connection": "XXX_DOCUMENTDB",
      "direction": "in"
    }
  ],
  "disabled": false
}
```

Important to notice is the `httpTrigger` with `"name": "input"`. `input` refers here to the parameter in the function definition. Also important is `"id": "{messageid}"`on the Cosmos db input binding. **Mind the curly braces around `messageid`!**

Also pay attention to the `messageDocument` parameter. It should be of type `dynamic`.

my run.csx:

```
using System.Net;

public class Input
{
    public string messageid { get; set; }
}

public static HttpResponseMessage Run(Input input, HttpRequestMessage req, dynamic messageDocument, TraceWriter log)
{
    string messageId = req.GetQueryNameValuePairs()
            .FirstOrDefault(q => string.Compare(q.Key, "messageid", true) == 0)
            .Value;
    log.Info($"Incoming parameter messageid = {input.messageid}");
    log.Info($"Incoming document = {messageDocument}");

    if (messageDocument != null)
    {
        return req.CreateResponse(HttpStatusCode.OK, (object)messageDocument);
    }
    else
    {
        return req.CreateResponse(HttpStatusCode.BadRequest);
    }
}
```


I added the Input class as a POCO to map its property `messageid` to the `id` property in the input binding for Cosmos db.
So, that is where the magic happens. 

If you know about a better way to do it, maybe without the POCO class, pls let me know!
