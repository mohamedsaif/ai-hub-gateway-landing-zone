# AI Hub Gateway Landing Zone
The AI Hub Gateway Landing Zone is a reference architecture that provides a set of guidelines and best practices for implementing a central AI API gateway to empower various line-of-business units in an organization to leverage Azure AI services.

## Overview
The AI Hub Gateway Landing Zone is designed to be a central hub for AI services, providing a single point of entry for AI services, and enabling the organization to manage and govern AI services in a consistent manner. 

![AI Hub Gateway Landing Zone](./assets/architecture-1-0-5.png)

## Features
The AI Hub Gateway Landing Zone provides the following features:

- **Centralized AI API Gateway**: A central hub for AI services, providing a single point of entry for AI services that can be shared among multiple use-cases in a secure and governed approach.
- **Seamless integration with Azure AI services**: Ability to just update endpoints and keys in existing apps to switch to use AI Hub Gateway.
- **AI routing and orchestration**: The AI Hub Gateway Landing Zone provides a mechanism to route and orchestrate AI services, based on priority and target model enabling the organization to manage and govern AI services in a consistent manner.
- **No master keys**: The AI Hub Gateway Landing Zone does not use master keys to access AI services, instead, it uses managed identities to access AI services while consumers can use gateway keys.
- **Private connecitity**: The AI Hub Gateway Landing Zone is designed to be deployed in a private network, and it uses private endpoints to access AI services.
- **Capacity management**: The AI Hub Gateway Landing Zone provides a mechanism to manage capacity based on requests and tokens.
- **Usage & charge-back**: The AI Hub Gateway Landing Zone provides a mechanism to track usage and charge-back to the respective business units with flexible integration with existing charge-back & data platforms.
- **Resilient and scalable**: The AI Hub Gateway Landing Zone is designed to be resilient and scalable, and it uses Azure API Management with its zonal redudancy and regional gateways which provides a scalable and resilient solution.
- **Full observability**: The AI Hub Gateway Landing Zone provides full observability with Azure Monitor, Application Insights, and Log Analytics with detailed insights into performance, usage, and errors.
- **Hybrid support**: The AI Hub Gateway Landing Zone approach the deployment of backends and gateway on Azure, on-premises or other clouds.

## Architecture components
The AI Hub Gateway Landing Zone consists of the following components:

### Main gateway compoenets
These are the critical components of the AI Hub Gateway Landing Zone that provides the capaiblities outlined above.

- **Azure API Management**: Azure API Management is a fully managed service that enables customers to publish, secure, transform, maintain, and monitor APIs.
- **Application Insights**: Application Insights is an extensible Application Performance Management (APM) service that provides critical insights on the gateway operational performance.
- **Event Hub**: Event Hub is a fully managed, real-time data ingestion service that’s simple, trusted, and scalable and it is used to stream usage and charge-back data to target data and charge back platforms.

### AI services
This is the Azure AI services that will be exposed through the AI Hub Gateway Landing Zone.

Examples of these service could include:

- **Azure OpenAI**: Azure OpenAI is a cloud deployment of cutting edge generative models from OpenAI (like ChatGPT, DALL.E and more).
- **Azure AI Search**: Azure AI Search is a cloud search service with built-in AI capabilities that enrich all types of information to help users identify and explore relevant content at scale (critical compoenet of RAG-based generative AI applications).
- **Azure Cognitive Services**: Azure Cognitive Services is a set of cloud-based services with REST APIs and client library SDKs available to help you build cognitive intelligence into your applications.

### Backend services
These are the backend services that will include your AI business logic and experiences.

You can host backend services on Azure, on-premises, or other clouds.

Examples of these services could include:
- **Azure Kubernetes Service**: Azure Kubernetes Service (AKS) is a managed container orchestration service, based on the open-source Kubernetes system, which is available on the Microsoft Azure public cloud.
- **Azure Container Apps**: Azure Container Apps is a fully managed serverless container service that enables you to run containers on Azure without having to manage the infrastructure.
- **Azure App Service**: Azure App Service is a fully managed platform for building, deploying, and scaling web apps.

Also in these backends, it is common to use **AI Orchestrator** framework like [Semantic Kernel](https://github.com/microsoft/semantic-kernel) and [Langchain](https://www.langchain.com/) to orchestrate sophisticated AI workflows and scenarios.

### Data and charge-back platforms
As part of the AI Hub Gateway Landing Zone, you will need to integrate with existing data and charge-back platforms to track usage and charge-back to the respective business units.

Examples of these platforms could include:
- **Cosmos DB**: Azure Cosmos DB is a fully managed NoSQL database for storing usage and charge-back data.
- **Azure Synapse Analytics**: Azure Synapse Analytics is an analytics service that brings together enterprise data warehousing and big data analytics.
- **Microsoft Fabric**: Microsoft Fabric is a cloud-based platform that provides a scalable, reliable, and secure infrastructure for building and managing data and analytics solutions.
- **PowerBI**: Power BI is a business analytics service by Microsoft. It aims to provide interactive visualizations and business intelligence capabilities with an interface simple enough for end users to create their own reports and dashboards.

## Deployment

Below is a high-level guide to deploy the AI Hub Gateway Landing Zone.

### Networking
For the AI Hub Gateway Landing Zone to be deployed, you will need to have/identify the following components:
- **Virtual network & subnet**: A virtual network to host the AI Hub Gateway Landing Zone.
    - APIM to be deployed in **internal mode** requires a subnet with /27 or larger with NSG that allows the critical rules.
    - **Private endpoints subnet(s)**: Private endpoints for the AI services to be exposed through the AI Hub Gateway Landing Zone. Usually a /27 or larger subnet would be sufficient.
- **Private DNS zone**: A private DNS zone to resolve the private endpoints.
    - Internal APIM reqlies on **private DNS** to resolve the APIM endpoints, so a Azure Private DNS zone or other DNS solution is required.
    - **Private endpoints DNS zone**: A private DNS zone to resolve the private endpoints for the connected Azure AI services.
- **ExpressRoute or VPN**: If you are planning to connect to on-premises or other clouds, you will need to have an ExpressRoute or VPN connection.
- **DMZ applicances**: If you are planning to expose backend and gateway services on the internet, you need to have a Web Application Firewall (like Azure Front Door & Application Gateway) and network filrewall (like Azure Firewall) to govern both ingress and egress traffic.

### Azure API Management (APIM)
APIM is the central component of the AI Hub Gateway Landing Zone. 

Recommended deployment of APIM to be in **internal mode** to ensure that the gateway is not exposed to the internet and to ensure that the gateway is only accessible through the private network.

**internal mode** requires a subnet with /27 or larger with NSG that allows the critical rules in addition to management public IP (with DNS label set)

This is a great starting point to deploy APIM in internal mode: [Deploy Azure API Management in internal mode](https://learn.microsoft.com/en-us/azure/api-management/api-management-using-with-internal-vnet?tabs=stv2)

### Application Insights
Application Insights is a critical component of the AI Hub Gateway Landing Zone, and it is used to monitor the operational performance of the gateway.

To deploy Application Insights, you can use the following guide: [How to integrate Azure API Management with Azure Application Insights](https://azure.github.io/apim-lab/apim-lab/6-analytics-monitoring/analytics-monitoring-6-2-application-insights.html) 

### Event Hub

Event Hub is used to stream usage and charge-back data to target data and charge back platforms.

To deploy Event Hub, you can use the following guide: [Logging with Event Hub]https://azure.github.io/apim-lab/apim-lab/6-analytics-monitoring/analytics-monitoring-6-3-event-hub.html)