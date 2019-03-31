---
description: ""
category: azure
tags: [azure, sql-server, c#, dotnet]
---
{% include JB/setup %}
A while ago I discovered that Azure Application insights can automatically stitch together the logs from the message consumer and message producer of Azure Service Bus messages (despite being saved in different Application Insights resources). This has been a long standing pain point for our message based architecture - now when we publish a message we have a sense of its outcome. Recently I showed this to a colleague - slightly impressed and he followed up with a predictable question: "how does it work?". This post describes the output of an embarassing amount of hours digging through icnomplete / out-of-date documentation and source code to answer this question.

