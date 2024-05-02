### Gateway routing strategies (APIM)

When it comes to GenAI APIs, a need for advanced routing strategies arises to manage the capacity and resiliency for smooth AI-infused experiences across multiple clients.

Setting these policies in APIM will allow for advanced routing based on the region and model in addition to the priority and throttling status.

Dimensions of the routing strategies include:
- **Global vs. regional**: Ability to route to traffic to different regional gateway might be a requirement to ensure low latency, high availability and data residency.
    - For example, if you have a global deployment of AI Hub Gateway, you might want to route traffic to the nearest gateway to the client, or route that traffic to a specific gateway based on regulatory requirements.
- **Model-based routing**: Ability to route to traffic based on requested model is critical as not all OpenAI regional deployments support all capabilities and versions.
    - For example, if you can have gpt-4-vision model that is only available in 2 regions, you might want to load balance traffic to these 2 regions only.
- **Priority based routing**: Ability to route traffic based on priority is critical to ensure that the traffic is routed to preferred region first and fall back to other deployments when primary deployment is not available.
    - For example, if you have a Provisioned Throughput Unit (PTU) deployment in certain region, you might want to route all traffic to that deployment to maximize its utilization and only fall back to a secondary deployment in another region when the PTU is throttling (this also should revert back to primary when it is available again).
- **Throttling fallback support**: Ability to take a specific route out of the routing pool if it is throttling and fall back to the next available route.
    - For example, if you have a OpenAI deployment that is throttling, AI Hub Gateway should be able to take it out of the routing pool and fall back to the next available deployment and register the time needed before it is available again you might want so it can be brought back into the pool.
- **Configuration update**: Ability to update the routing configuration without affecting the existing traffic is critical to allow for rolling updates of the routing configuration.
    - For example, if you have a new OpenAI deployment that is available, you might want to update the routing configuration to include it and allow for new traffic to be routed to it without affecting the existing traffic (and in also support rolling back certain update when needed).

In the [src/apim/policies](/src/apim/oa-fragments) folder, you will find the policy fragments that you can use to apply to the OpenAI API.

I've built my routing strategy based the great work of [APIM Smart Load Balancing](https://github.com/andredewes/apim-aoai-smart-loadbalancing/tree/main), it is worth checking out.

I've built on top of that additional capabilities to make the solution address all the needs outlined above for a robust and reliable AI routing engine.

Implementation details are as follows:
- **Clusters (model based routing)**: it is a simple concept to group multiple OpenAI endpoints that support specific OpenAI deployment name. 
    - This to support model-based routing
    - For example, if the model is gpt-4 and it exists only in 2 regions, I will create a cluster with these 2 endpoints only. On the other hand, gpt-35-turbo exists in 5 regions, I will create a cluster with these 5 endpoints.
    - In order for this routing to work, OpenAI deployment names across regions must use the same name as I rely on the URL path to extract the direct deployment name which then results in specific routes to be used.
- **Routes**: It is json array that include all OpenAI endpoints with metadata.
    - Each cluster will reference supported route from this json array
    - Each route will have a friendly name, location, priority, and throttling status.
- **Clusters and routes caching**: using APIM cache to store clusters and routes to allow it to be shared across multiple API calls contexts.
    - **Configurations update**: Using API revision part of the caching key to allow for rolling updates of the clusters and routes through:
        - Creating new API revision with updated clusters and routes
        - Updating the API revision to be current (which will result in immediate creation of new cache entry with the updated clusters and routes)
        - API revision number is part of the cache key for both clusters and routes.
        - If configuration roll back is critical, you might want to add the routing policies directly in OpenAI API - All Operations policy scope (as policy fragments don't support revisions).
    - **Multi-region support**: Each clusters array will be stored with region name as part of the cache key to allow for multi-region support.

Based on this implementation, APIM should be able to do advanced routing based on the region and model in addition to the priority and throttling status.

Having revision number as part of the cache key will allow for rolling updates of the clusters and routes.

Also at any given time, you will have different cached routes that represent different models/region, and based on the incoming request, you can route to the correct OpenAI endpoint.

Let's have a look at the policy fragments components:

#### oai-clusters-lb-configuration-in-policy.xml
This inbound policy fragment contains the main clusters and routes configurations.

Let's have a look at the configuration components:

- **Routes**: a json array that include the routes to various Azure OpenAI endpoints.
```csharp
JArray routes = new JArray();
routes.Add(new JObject()
{
    { "name", "EastUS" },
    { "location", "eastus" },
    { "backend-id", "openai-backend-0" },
    { "priority", 1},
    { "isThrottling", false }, 
    { "retryAfter", DateTime.MinValue } 
});

routes.Add(new JObject()
{
    { "name", "NorthCentralUS" },
    { "location", "northcentralus" },
    { "backend-id", "openai-backend-1" },
    { "priority", 1},
    { "isThrottling", false },
    { "retryAfter", DateTime.MinValue }
});

routes.Add(new JObject()
{
    { "name", "EastUS2" },
    { "location", "eastus2" },
    { "backend-id", "openai-backend-2" },
    { "priority", 1},
    { "isThrottling", false },
    { "retryAfter", DateTime.MinValue }
});
```
- **Clusters**: a json array that include the clusters to various Azure OpenAI endpoints.
```csharp
JArray clusters = new JArray();
clusters.Add(new JObject()
        {
            { "deploymentName", "gpt-35-turbo" },
            { "routes", new JArray(routes[0], routes[1]) }
        });

clusters.Add(new JObject()
        {
            { "deploymentName", "embedding" },
            { "routes", new JArray(routes[0], routes[1]) }
        });

clusters.Add(new JObject()
        {
            { "deploymentName", "gpt-4" },
            { "routes", new JArray(routes[0]) }
        });

clusters.Add(new JObject()
        {
            { "deploymentName", "dall-e-3" },
            { "routes", new JArray(routes[0]) }
        });
```
- **Caching**: caching the clusters and routes to allow it to be shared across multiple API calls contexts.
```xml
<cache-store-value key="@("oaClusters" + context.Deployment.Region + context.Api.Revision)" value="@((JArray)context.Variables["oaClusters"])" duration="60" />

<cache-store-value key="@(context.Request.MatchedParameters["deployment-id"] + "Routes" + context.Deployment.Region + context.Api.Revision)" value="@((JArray)context.Variables["routes"])" duration="60" />
```

#### oai-clusters-lb-configuration-be-policy.xml
This backend policy fragment contains the main routing logic for the configured inbound policy above.

It selects the available routes based on model, region and API revision and provide the smart load balancing capabilities:
- Priority based routing:
    - Like if you have a cluster with 3 routs, 2 with priority 1 and 1 with priority 2, the gateway will always randomly select one of the 2 routes with priority 1 first and fall back to priority 2 if the first 2 routes are not available (is throttling).
- Throttling support:
    - Ability to take a specific route out of the routing pool if it is throttling and fall back to the next available route.
    - Activate the throttling route after a specific time (retryAfter) to allow for the route to be available again.

For more elaborate description of this routing, refer to the original [APIM Smart Load Balancing](https://github.com/andredewes/apim-aoai-smart-loadbalancing/tree/main) implementation.

#### oai-usage-eventhub-out-policy.xml
This outbound policy fragment contains the main usage tracking logic for the configured inbound policy above.

It sends the usage data to the configured Event Hub to allow for usage tracking and charge-back.

To use this policy, you need first to configure the Event Hub logger connection string and name.
```ps1
# API Management service-specific details
$apimServiceName = "apim-ai-gateway"
$resourceGroupName = "rg-ai-gateway"

# Event Hub connection string
$eventHubConnectionString = "Endpoint=sb://<EventHubsNamespace>.servicebus.windows.net/;SharedAccessKeyName=<KeyName>;SharedAccessKey=<key>"

# Create logger
$context = New-AzApiManagementContext -ResourceGroupName $resourceGroupName -ServiceName $apimServiceName
New-AzApiManagementLogger -Context $context -LoggerId "usage-eventhub-logger" -Name "usage-eventhub-logger" -ConnectionString $eventHubConnectionString -Description "Event Hub logger for OpenAI usage metrics"
```

Using this policy, you will have records like the following (I used CosmosDb to store these metrics from Event Hub through Stream Analytics job):

```json
{
    "id": "chatcmpl-91p2WwO4gvev3KSpwDwWjvdMsEpDs",
    "timestamp": "2024-03-12T05:32:32.0000000Z",
    "appId": "0000000-0000-4f5f-8ccf-8287272b09ad",
    "subscriptionId": "master",
    "productName": "AI-Marketing",
    "targetService": "chat.completion",
    "model": "gpt-35-turbo",
    "routeUrl": "https://REPLACE1.openai.azure.com/openai",
    "routeLocation": "swedencentral",
    "routeName": "SwedenCentralAzureOpenAI",
    "promptTokens": 9,
    "responseTokens": 10,
    "totalTokens": 19,
    "EventProcessedUtcTime": "2024-03-12T05:33:45.5528316Z",
    "PartitionId": 0,
    "EventEnqueuedUtcTime": "2024-03-12T05:32:32.8800000Z",
    "deploymentName": "gpt-35-turbo"
}
```

Based on these records, I've created the following PowerBI dashboard to track the usage and charge-back:

![PowerBI dashboard](./assets/powerbi-usage-dashboard.png)

#### oai-blocked-streaming-in-policy.xml
This inbound policy fragment that prevent streaming requests to the OpenAI API.

Currently streaming has 2 challenges when it comes to charge back and usage tracking:
- Current approach to usage metrics logging do not support streaming requests due to conflict with response buffering (which result in 500 error), so you can't use ```log-to-eventhub``` policy.
- OpenAI streaming requests do not provide usage metrics in the response as it stands today (maybe it will change in the future).

APIM is perfectly fine to proxy streamed backends, but usage metrics will not be captured.

One solution to this is to use an app as backend to log the usage metrics and charge-back and proxy the streaming requests.

This app will rely on a token counting SDK to manually calculate the tokens and ship them to Event Hub when steam is done.

Check out one implementation for this on [enterprise-azure-ai](https://github.com/Azure/enterprise-azureai) with an AI Proxy app that can do that.

I'm working on adopting this app so it will be only used for streaming requests (currently it is designed to do both streaming and non-streaming requests in addition to having the routing logic).

There are few ways to handle this, one of them is to use an app as backend to log the usage metrics and charge-back and proxy the streaming requests.

#### Capacity management
In OpenAI calls, tokens are used to manage capacity and rate limits.

Currently APIM natively support rate limiting on the number of requests per time window, but we can leverage that to repurpose it to manage capacity based on tokens.

APIM policy [rate-limit-by-key](https://docs.microsoft.com/en-us/azure/api-management/policies/rate-limit-by-key) can be used to manage capacity based on tokens.

```xml

<!-- Rate limit on TPM (Outbound Policy) -->
<!-- Note: this policy is designed to be integrated with other APIM policies in this guide -->
<rate-limit-by-key calls="5000" renewal-period="60" 
    counter-key="@(String.Concat(context.Subscription.Id,"tpm"))" 
    increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)" 
    increment-count="@(((JObject)context.Variables["responseBody"]).SelectToken("usage.total_tokens")?.ToObject<int>() ?? 0)" 
    remaining-calls-header-name="remainingTPM" total-calls-header-name="totalTPM" />

```

Above policy will limit the number of tokens to 5000 per minute based on the total tokens used in the response (that is why it is an outbound policy).

This policy will also add 2 headers to the response to indicate the remaining tokens and the total tokens used.

This will allow the client calling the api to know how many tokens are remaining and how many tokens are used.

You can also combine token rate limiting with request rate limiting to provide a more robust capacity management.

```xml

<!-- Rate limit on RPM -->
<rate-limit-by-key calls="5" renewal-period="60" counter-key="@(String.Concat(context.Subscription.Id,"rpm"))" increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)" remaining-calls-header-name="remainingRPM" total-calls-header-name="totalRPM" />

```

One more capacity control to use is the [quota-by-key](https://docs.microsoft.com/en-us/azure/api-management/policies/quota-by-key) policy.

```xml
<!-- Quota limit on requests per 5 mins -->
<quota-by-key calls="100" renewal-period="300" counter-key="@(context.Subscription.Id)" increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)" />

```
Above quota policy will limit the number of requests to 100 per 5 minutes.

My recommendation is to use only the minimum required capacity management policies to avoid over-complicating the solution (for example, token limit only can be sufficient in some cases).

>NOTE: I believe native policy support is coming to APIM soon, but for now, you can use the custom rate limiter to manage capacity based on tokens.