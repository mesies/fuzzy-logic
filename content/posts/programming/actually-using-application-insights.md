---
title: "Actually using Application Insights"
date: 2024-05-19T21:07:19+02:00
tags: [telemetry]
draft: true
---

So, we just deployed to production for the first time our new shiny product, since my company is a microsoft shop we used blazor + dotnet core. We started testing our 3rd party integrations and something was not working as intended. Meetings were called to find out what system sent what etc. Since our team has an aggressive telemetry strategy, we could tell instantly what we sent, when, and what the response was.

The conversation went something like this:
Me : We know that we got a 500 response from application insights
A PM from another team : You guys use that?? Its very expensive. I envy your budget.

Our application has medium traffic last month we got a 26$ bill for production

Don't get me wrong, someone could very well read the logs, but in a production scenario, this is not only extremely inconvenient but also a very very bad idea.
 In the modern world, where application live primarily in the cloud, when horizontal scaling, multiple services, are a reality, how could someone find a log about an incoming http call? Should they search the logs of instance 1, instance 2? An error prone procedure. What about searching for keywords, datetime ranges? I won't even go there.



1 Create a preprocessor to ignore hangfire sql calls
  Out dev application insights used 20$/month, so naturally since dev environment did not have the traffic to warrant such high volume of logs
  we investigated. We found out that most data stored in applciation insights were sql dependency events relating to Hangfire.  
2 Create a preprocessor to ignore calls that have no value, e.g. robots.txt etc.
3 Create a preprocessor to add trace identifier to all telemetry items, yes yes all items have already operationId, but if you use traces for your logs

-- insert code example?
-- maybe redo this with opentelemetry?