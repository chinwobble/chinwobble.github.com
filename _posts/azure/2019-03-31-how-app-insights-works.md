---
layout: post
category: azure
tags: [azure, sql-server, c#, dotnet]
title: how application insights correlates telemetry
---
{% include JB/setup %}
A while ago I discovered that Azure Application insights can automatically stitch together the logs from the message consumer and message producer of Azure Service Bus messages (despite being saved in different Application Insights resources). This has been a long standing pain point for our message based architecture - now when we publish a message we have a sense of its outcome. Recently I showed this to a colleague - slightly impressed and he followed up with a predictable question: "how does it work?". This post describes the output of an embarassing amount of hours digging through incomplete / out-of-date documentation and source code to answer this question.

The first thing to understand is how does Application Insights (in the case of dotnet at least) know about your application dependencies such as SQL calls and http requests. Naturally the [offical docs](https://docs.microsoft.com/en-us/azure/azure-monitor/app/auto-collect-dependencies) provides little detail on HOW dependency collection works. The one clue is highlighted below:
```
Monitoring works by using byte code instrumentation around select methods or based on DiagnosticSource callbacks (in the latest .NET SDKs) from the .NET Framework. Performance overhead is minimal.
```
Since we're dealing with ASP.NET Core we can safely assume we're using *DiagnosticSource*.

This takes us the [DiagnosticSource User's Guide](https://github.com/dotnet/corefx/blob/master/src/System.Diagnostics.DiagnosticSource/src/DiagnosticSourceUsersGuide.md). The saliant point of the article is that in dotnet has a feature called `DiagnosticSource` that allows libraries to publish messages such as instrumentation information that any other library can listen to and even modify. Application Insights can automatically track dependencies [to supported targets](https://docs.microsoft.com/en-us/azure/azure-monitor/app/auto-collect-dependencies) since these libraries implement DiagnosticSource publishers - using this hook, it can read messages, log them and enrich with extra data.

For instance, *System.Net.Http.HttpClient* will automatically add a [DiagnosticsHandler](https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/DiagnosticsHandler.cs#L54) that is responsible for publishing diagnostics about when outbound requests have started and ended. If a diagnostic listener is subscribed, it can access the entire HttpRequestMessage and HttpResponseMessage objects. This provides a powerful hook to log and enrich outbound messages and their repsonses. Some more digging through the
[Microsoft.ApplicationInsights.Web.DependencyCollector source code](https://github.com/Microsoft/ApplicationInsights-dotnet-server/blob/develop/Src/DependencyCollector/Shared/HttpCoreDiagnosticSourceListener.cs#L336) reveals that application insights uses these hooks to attach additional http headers that the target service can parse and correlate it to a parent operation. Astutate readers will notice, in typical Microsoft fashion, the code base is littered with deprecation and legacy comments. Further, there no less than three formats for tracing headers in the code base: `Legacy Headers`, `W3C Distributed Tracing Prosposal` and `Microsoft's Http Correlation Protocol`.

With this background knowledge on DiagnosticSource out of the way we can now proceed with a concrete example of how everything works:
1. A user creates a httpRequest to a WEB API. This is the root operation and contains an id.
2. The application publishes a service bus message. Just before publishing application insights will hook into the event for publishing and add an extra header called `Diagnostic-Id` to the service bus message. This value is a hierarchical id that has the dependency and root operation id encoded in it. In other words, anyone reading this `Diagnostic-Id` can trace back to step 1 or step 2. The service bus `messageId` is also logged as a custom dimension.
3. Another service will consume the message. If application insights is installed on the consuming app, `Microsoft.ApplicationInsights.Web.DependencyCollector` will read the message and look for the `Diagnostic-Id` header value. The processing of the service bus operation is logged a request telemetry. However we it uses the received `Diagnostic-Id` as its operation parent. This means that stpe 3 is a child of step 2.

Example by data:

step| telemetry-id  | parent-id  | operation-id | name  |
1   | 111         | 111      | 111        | inbound request to api|
2   | 111.12356.12356_1.      | 111.12356| 111|send a message to application inisghts with the telemetry-id set as the `diagnostics-id` header|
3   | 111.12356.12356_1.567              |   111.12356.12356_1.| 111||


Resources:
https://github.com/dotnet/corefx/blob/master/src/System.Diagnostics.DiagnosticSource/src/DiagnosticSourceUsersGuide.md



