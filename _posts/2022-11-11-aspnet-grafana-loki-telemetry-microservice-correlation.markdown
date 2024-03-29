---
layout: default
title: "Adding correlation metadata to our ASP.Net Core application logs"
description: "Configuration of Serilog to link distributed application logs in a microservice architecture"
excerpt: "Let correlate our logs from multiple microservices with a correlation id with Serilog (and optionally Loki)"
date: 2022-11-09 23:52:00 +0200
categories: [AspNetCore]
tags: [AspNetCore, C#, Loki, Serilog, Microservices]
---

# Adding a correlation id and other metadata to our logs 

In the previous post, we were able to send our logs to Loki using Serilog. In a monolithic application, it may be overkill,
but in a microservice architecture, centralizing and linking our logs is mandatory to exploit them efficiently.

To do so, we will enrich our logs with a correlation id and some other metadata, and forward them to subsequent http requests.

I'll suppose you are using Loki and have followed the previous article, but most of the content of the current post is 
applicable to any Serilog sink.

## Wording

I'll call a module a microservice application that does not perform any http requests, and a gateway a service that 
delegates to at least one module.

When I say field, I mean a piece of information included in the log. A label is a field that is indexed by Loki. 
If you aren't using Loki, you can consider both terms as synonyms.  

## Getting / defining our additional fields

First, here is my class for getting or defining my fields. 

The CorrelationId is a unique identifier, passed to the subsequent http requests.

The RootInitiator defines the first item in the chain, either the frontend application, the first gateway, or the module 
itself when called through Swagger for instance, or for a console application.

The AppName is the name of my startup project, but you can of course use another name.

```cs
using System.Diagnostics;
using System.Reflection;

namespace Ari.LokiMicroservices.Logs;

public static class LogAndTraceMetadata
{
    public const string correlationIdKey = "x-correlation-id";
    public const string rootInitiatorKey = "x-root-initiator";

    public static string GetCorrelationId(HttpContext httpContext)
    {
        string? correlationId = null;
        if (httpContext.Request.Headers.TryGetValue(correlationIdKey, out var values))
        {
            correlationId = values.FirstOrDefault();
        }

        return correlationId ?? Activity.Current?.RootId ?? httpContext.TraceIdentifier;
    }

    public static string GetRootInitiator(HttpContext httpContext)
    {
        string? rootInitiator = null;
        if (httpContext!.Request.Headers.TryGetValue(rootInitiatorKey, out var initiatorValues))
        {
            rootInitiator = initiatorValues.FirstOrDefault();
        }
        return rootInitiator ?? GetAppName();
    }

    public static string GetAppName()
    {
        return Assembly.GetCallingAssembly().GetName().Name!;
    }
}
```

Now, we need to enrich our logs with these fields:

```cs
using Serilog.Core;
using Serilog.Events;

namespace Ari.LokiMicroservices.Logs;

public class CorrelationIdEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _contextAccessor;

    public CorrelationIdEnricher() : this(new HttpContextAccessor())
    {
    }

    internal CorrelationIdEnricher(IHttpContextAccessor contextAccessor)
    {
        _contextAccessor = contextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        logEvent.AddOrUpdateProperty(new LogEventProperty("App", new ScalarValue(LogAndTraceMetadata.GetAppName())));

        if (_contextAccessor.HttpContext == null)
            return;

        logEvent.AddOrUpdateProperty(new LogEventProperty("CorrelationId", new ScalarValue(LogAndTraceMetadata.GetCorrelationId(_contextAccessor.HttpContext!))));
        logEvent.AddOrUpdateProperty(new LogEventProperty("RootInitiator", new ScalarValue(LogAndTraceMetadata.GetRootInitiator(_contextAccessor.HttpContext!))));
    }
}
```

If you are using the Loki sink, as described in the previous article, you can remove the labels property from your 
appsettings, as the App field is now defined from the enricher:

```json
{
          "labels": [
            {
              "key": "App",
              "value": "My Application"
            }
          ]
}
```

And then, we have to register our enricher to Serilog configuration, but also to expose the HttpContext so that we can
grab the values sent by the previous application of the chain.

```cs
// Program.cs

builder.Services.AddHttpContextAccessor();

var logger = new LoggerConfiguration()
  .ReadFrom.Configuration(builder.Configuration)
  .Enrich.With<CorrelationIdEnricher>()
  .CreateLogger();
builder.Logging.ClearProviders();
builder.Logging.AddSerilog(logger);
```

Now, we're done with the logger configuration. 

## Transmitting our metadata to the subsequent requests

First, you'll have to install the *Microsoft.AspNetCore.HeaderPropagation* nuget package.

If your project is a module that makes no HTTP requests, you can stop here, but for a gateway, you still have to forward
its log context to its dependencies. To do so, we have to set the header propagation mechanism up:

```cs
// Program.cs

builder.Services.AddHeaderPropagation(options => {
    options.Headers.Add("x-correlation-id", context => LogAndTraceMetadata.GetCorrelationId(context.HttpContext));
    options.Headers.Add("x-root-initiator", context => LogAndTraceMetadata.GetRootInitiator(context.HttpContext));
});

//[...]

app.UseHeaderPropagation();
```

And for each of your HttpClient, don't forget to enable the header propagation like this :

```cs
services.AddHttpClient("NamedClient", c =>
    {
        c.BaseAddress = new Uri(Options.Url);
    }).AddHeaderPropagation();
```

Now, your fields should be transmitted properly, and they should appear in your logs.

## Indexing the fields

We are sending additional fields to Loki, but they are only part of the log context, and are not indexed. 
You cannot query them directly, as Loki has to parse the log content first. Add the fields you want to the propertiesAsLabels 
property in the appsettings in order to index them. 

Use as few fields as you can, and avoid indexing dynamic fields, such as the correlation id, to keep Loki fast. 
I currently only index App and RootInitiator.

We are now able to query the logs using LogQL. On the next article, I'll share a basic Grafana dashboard to search in the
logs with ease.