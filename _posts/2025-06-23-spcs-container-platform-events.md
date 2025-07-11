---
title: 'Debugging SPCS container crashes with Platform Events'
date: 2025-06-23 10:10:00-08:00
excerpt: When deploying containers in Snowflake via Snowpark Container Services (SPCS), the ability to debug container failure conditions is key as workloads mature and go to production. To address these scenarios, SPCS infrastructure now emits Platform Events around container status. 
tags:
- Snowflake
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/AW3xNuOUe-U?si=n3QS0yNnqHWiEysl" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

When deploying containers in Snowflake via Snowpark Container Services (SPCS), the ability to debug container failure conditions is key as workloads mature and go to production. [Application logs](2025-05-12-spcs-observability-trail.md) can offer insights into the stack trace of a failure, but that still leaves some gaps in the experience:
- **Out of memory errors** - containers get shut down (OOMkilled) when they exceed the allowed amount of allocated memory
- **Restart loops** - when a container is running as a Service, it will automatically get restarted if it exits. In certain cases, containers will start failing repeatedly with the same error, and will repeatedly crash and restart (crashloop). This in turn will waste resources and prevent the Compute Pool from suspending, potentially leading to unexpected charges. 
- **Failure to start** - sometimes containers fail because the underlying image is no longer available, or a readiness probe is failing, and the container gets stuck in a `PENDING` state.
- **Alerting** - parsing through logs is great for reactive analysis, but it isn't possible to set up alerts to be proactively notified of failures 

To address these scenarios, SPCS infrastructure now emits Platform Events around container status. These are events automatically being emitted by the underlying hosting infrastructure. The full list of events is [documented here](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#supported-events), but some highlighted conditions include: 
- Readiness probe is failing
- Compute pool node(s) are being provisioned
- Failed to pull image
- Provided image name uses an invalid format
- Container was OOMKilled due to resource usage

As with other telemetry types, Platform Events are stored in the global Event Table for the account, and are accessible via SQL (`RECORD_TYPE = 'EVENT' AND SCOPE:"name" = 'snow.spcs.platform'`) or via Snowsight UI (use the Record filter to find Events, or leave unset to see both logs and events):

![](/images/posts/spcs-container-platform-events.png)


### Next steps
📚 Review our  [documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#accessing-platform-events)<br/>
📣 Let us know what you think by responding to this post