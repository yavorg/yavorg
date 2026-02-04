---
title: 'Achieving High Availability and Safe Deploys in Snowpark Container Services'
date: 2026-02-03 10:00:00-08:00
featured_image: '/images/posts/spcs-gateway.png'
excerpt: Today we announce the General Availability of the Ingress Gateway, providing advanced traffic management capabilities in SPCS. This feature enables stable URLs and traffic splits as part of a mature deployment strategy. Combined with compute support for placement groups, this feature also offers a complete toolkit for building mission-critical, high-availability applications.
tags:
- Snowflake
---

When we first launched **Snowpark Container Services (SPCS)**, our goal was simple: bring the power of containers directly to your data. Since then, we've seen developers build everything from complex machine learning pipelines to interactive web applications. 

As these workloads move into production, two requirements have become paramount: **resilience** and **control**. Developers need to know their apps can survive infrastructure hiccups, and they need a way to roll out updates without customer disruption. 

Today, we are thrilled to announce the **General Availability of the Ingress Gateway**, providing advanced traffic management capabilities in SPCS. This feature enables stable URLs and traffic splits as part of a mature deployment strategy. Combined with compute support for Placement groups, this feature also offers a complete toolkit for building mission-critical, high-availability (HA) applications.


### The evolution of ingress: introducing Gateways

In the early days of SPCS, ingress was tied directly to a specific service. While functional, this created a "naming" challenge. If you needed to rotate a service or replace it with a new version, the URL often changed, creating a headache for downstream consumers and requiring manual updates to DNS or application configurations.

**Gateways** solve this by decoupling the ingress endpoint from the underlying service. A Gateway provides:
* **Stable URLs**: Once created, a Gateway’s hostname remains constant for its entire lifetime, even if you point it to different services behind the scenes.
* **Traffic Splitting**: You can distribute traffic across multiple services based on weights. This is perfect for **blue-green deployments** or canary testing.
* **Automatic Failover**: Gateways can automatically redirect traffic if a primary endpoint becomes unhealthy or its compute pool is suspended.


### Enabling safe deployment practices

With a Gateway, you keep a single, stable hostname even when you recreate or rotate the underlying service for scaling, patching, or migrations. This feature lets SDKs, partner integrations, allow lists, dashboards, bookmarks, and monitoring continue working without DNS updates, cache churn, or credential changes. You can also run blue/green releases for safe rollouts: run the new version alongside the current one, shift a small percentage of traffic, watch errors and latency, then ramp to 100% or roll back instantly by tweaking weights—no disruptive cutovers.

You can define a Gateway using SQL to split traffic between a production service and a new version:

```sql
CREATE OR REPLACE GATEWAY my_app_gateway
FROM SPECIFICATION $$
spec:
  type: traffic_split
  split_type: custom
  targets:
    - type: endpoint
      value: my_db.my_schema.service_v1!api
      weight: 90
    - type: endpoint
      value: my_db.my_schema.service_v2!api
      weight: 10
$$;
```
For full syntax details, see the [CREATE GATEWAY documentation](https://docs.snowflake.com/en/sql-reference/sql/create-gateway).


### Infrastructure isolation with placement groups

While Gateways manage *traffic*, **Placement groups** manage *where* your code actually runs. High Availability isn't just about software; it's about physical hardware.

[Placement groups](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-compute-pool#compute-pool-placement) are an existing capability for SPCS compute pools that allow you to dictate the physical distribution of your nodes. By deploying your application across two different compute pools assigned to different Placement Groups (e.g., Group A and Group B), you ensure that your service is distributed across different infrastructure domains or Availability Zones. 

If a specific rack or zone experiences a localized outage, your service remains alive on the other "side" of the house.

So, how do you actually build an HA service? It's a three-step process:

1.  **Redundant infrastructure**: Create two compute pools, each in a different `PLACEMENT_GROUP`.
2.  **Redundant services**: Deploy your service (or separate versions of it) to both compute pools.
3.  **Unified entry point**: Create a Gateway that points to both services.

Here is what that SQL looks like in practice:

```sql
-- Step 1: Create Compute Pools in different groups
CREATE COMPUTE POOL pool_zone_1 
  MIN_NODES = 1 MAX_NODES = 5 
  INSTANCE_FAMILY = CPU_X64_S 
  PLACEMENT_GROUP = A;

CREATE COMPUTE POOL pool_zone_2 
  MIN_NODES = 1 MAX_NODES = 5 
  INSTANCE_FAMILY = CPU_X64_S 
  PLACEMENT_GROUP = B;

-- Step 2: Deploy your services
-- (Assuming service_a is on pool_zone_1 and service_b is on pool_zone_2)

-- Step 3: Create the Gateway for 50/50 load balancing
CREATE OR REPLACE GATEWAY production_gateway
FROM SPECIFICATION $$
spec:
  type: traffic_split
  split_type: custom
  targets:
    - type: endpoint
      value: my_db.my_schema.service_a!api
      weight: 50
    - type: endpoint
      value: my_db.my_schema.service_b!api
      weight: 50
$$;
```

With this configuration, Snowflake handles the heavy lifting of load balancing and health monitoring. If `service_a` goes down, the Gateway automatically routes traffic to `service_b`.


### What's Next?

The General Availability of Gateways is a major milestone in making Snowflake the best place to run enterprise-grade applications. We are continuing to invest in more granular networking controls and even deeper integration with Snowflake's security model.

* 📺 **Watch the Demo**: Check out this [walkthrough video](https://youtu.be/Fxu2PMhfiqo) to see Gateways in action.
* 📚 **Read the Docs**: Explore the official documentation for [Gateways](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/gateway) and [Compute Pool Placement](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-compute-pool#compute-pool-placement).
* 🛠️ **Get Started**: These features are available today for all Snowflake accounts supporting Snowpark Container Services.

We can't wait to see the resilient, high-scale applications you build next!
