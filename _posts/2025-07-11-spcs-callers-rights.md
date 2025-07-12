---
title: 'Building apps that access Snowflake as the caller: a Gradio chatbot for Cortex Analyst'
date: 2025-07-11 10:10:00-08:00
featured_image: '/images/posts/spcs-rcr.png'
excerpt: We're now making it easier than ever to build multi-user containerized apps that leverage Snowflake's RBAC model. Check out this example of a Gradio chatbot using Cortex Analyst, while using the user/caller's rights. 
tags:
- Snowflake
---

Imagine you are building a chatbot, dashboard, or any other web-based frontend backed by Snowflake-hosted data. You have a great choice out of the box with [Streamlit in Snowflake](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit), but what if you want to port existing code, or use React, Gradio, or another frontend framework? In that case, deploying a a containerized app to Snowpark Container Services (SPCS) can provide a quick and familiar experience, where cloud-native devs feel right at home. 

In this scenario, one key piece you would want to leverage is your existing investment in Snowflake's RBAC model. In other words, when building an app, you expect users to only be able to access the data they have been granted the privileges to, a concept we refer to as "caller's rights". Infact, this is a key reason to move apps and dashboards inside the Snowflake security boundary - **seamlessly extending the trust model you've so meticulously crated to your apps as well**. Let's take a look at how to leverage this concept in a web-based chatbot, built using Gradio, that uses Cortex Analyst to allow natural-language querying of revenue data. 

<iframe width="560" height="315" src="https://www.youtube.com/embed/OmW4-DR5Ivk?si=arXEVJP7SJIP396I" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### App overview

We start with the [Getting Started with Cortex Analyst](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit) quickstart as a starting point, and then extend it with a Gradio frontend. 

1. The quickstart begins with [setting up](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/#1) the environment as well as a `cortex_analyst_demo.revenue_timeseries` schema containing a few revenue tables, and [populating them](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/#2) with sample data. 
1. A [Cortex Analyst semantic model](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/#5) is made available, which facilitates the mapping of tabular data to semantic concepts. More on the Cortex semantic layer [here](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-model-spec)
1. Next, a [Cortex Search service](https://quickstarts.snowflake.com/guide/getting_started_with_cortex_analyst/#3) is created to improve the accuracy of the Cortex Analyst SQL query generation. 
1. Set your user's default role to `CORTEX_USER_ROLE`

One key thing to point out is that access to the data and semantic layer is granted only to the role `CORTEX_USER_ROLE`, as you would in a real-world setting where only certain Snowflake users would have access to revenue data. Now that we have the data and semantic layer ready, let's look at how to put together a chatbot frontend for Cortex Analyst using Gradio. 

1. Run the SQL commands in [create_service.sql](https://github.com/Snowflake-Labs/spcs-templates/blob/main/rcr/create_service.sql) to create a `app_db.public.repo` image repository in your account. 
1. Build a container image for the [app](https://github.com/Snowflake-Labs/spcs-templates/tree/main/rcr/ui) and push it to the above repository, following [these steps](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/tutorials/tutorial-1).
1. Find the `analyst_ui` service you just created in the **Services and Jobs** tab in Snowsight, and navigate to the `chat` endpoint. You should be prompted to log in and then see the chatbot UI

Pay close attention to the section of lines similar to `GRANT CALLER ... IN SCHEMA cortex_analyst_demo.revenue_timeseries TO ROLE service_user_role`. The Gradio app will run as `SERVICE_USER_ROLE`, however that doesn't have direct privileges to access the revenue data the way `CORTEX_USER_ROLE` has. So the app will have to act on behalf of `CORTEX_USER_ROLE` and the list of privileges assigned via `GRANT CALLER` are the ones that the service can use. To confirm this explicitly, we can run `SHOW GRANTS ON DATABASE cortex_analyst_demo` and we should only see the following:
```
+-------------------------------+-----------+------------+---------------------+------------+
| created_on                    | privilege | granted_on | name                | granted_to |
+-------------------------------+-----------+------------+---------------------+------------+
| 2025-05-07 21:51:08.466 +0000 | OWNERSHIP | DATABASE   | CORTEX_ANALYST_DEMO | ROLE       |
+-------------------------------+-----------+------------+---------------------+------------+
```

### Addressing the limitations of owner's rights
By default, SPCS services operate under the Service Owner role when interacting with the Warehouse, Cortex, and other Snowflake components. While functional, this approach presents several challenges:
- *Overly broad privileges:* The Service Owner role often requires a union of all privileges needed by the application, leading to broader access than necessary.
- *Inconsistent experience for users:* Users might experience inconsistent privilege levels when accessing data through SPCS Services compared to direct access.
- *Difficulty in multi-user app implementation:* Implementing multi-user applications on SPCS becomes complex due to these privilege limitations.

The **caller's rights feature, now generally available,** directly addresses these issues by allowing the service to assume the roles of the user invoking it through a web interface. This brings a host of advantages:
- *Enhanced security:* The Service Owner role can now maintain a minimal set of privileges, aligning with security best practices.
- *Consistent User Experience:* User privileges remain consistent whether they are using a SPCS application or directly accessing the Warehouse.
- *Simplified Multi-User Application Development:* Building multi-user applications with role-specific behaviors (e.g., finance users accessing only finance data) becomes significantly easier.

This capability unlocks multi-user and multi-tenant application scenarios, allowing users to leverage their investment in Snowflake RBAC more effectively and preventing the security anti-patterns associated with granting excessive permissions to a service role.

### How it works
Implementing caller's rights involves a few key steps:

1. You must explicitly grant "caller grants" to the service's owner role. These grants allow the service certain privileges only when the caller has those privileges. Excerpting from the code sample:

```sql
-- cortex_user_role owns the database and schema
GRANT OWNERSHIP ON SCHEMA cortex_analyst_demo.revenue_timeseries TO ROLE cortex_user_role;
GRANT OWNERSHIP ON DATABASE cortex_analyst_demo TO ROLE cortex_user_role;

-- service_user_role only has caller grants 
GRANT CALLER USAGE ON DATABASE cortex_analyst_demo TO ROLE service_user_role;
GRANT INHERITED CALLER USAGE ON ALL SCHEMAS IN DATABASE cortex_analyst_demo TO ROLE service_user_role;
GRANT INHERITED CALLER USAGE,READ ON ALL STAGES IN SCHEMA cortex_analyst_demo.revenue_timeseries TO ROLE service_user_role;
GRANT INHERITED CALLER SELECT ON ALL TABLES IN DATABASE cortex_analyst_demo TO ROLE service_user_role;

```

2. In your SPCS service specification, you need to set `executeAsCaller` to true within the `securityContext` under `capabilities`.

```yaml
capabilities:
  securityContext:
    executeAsCaller: true
```

3. SPCS inserts an `Sf-Context-Current-User-Token` header in every incoming request to the application container. Your application code needs to construct a login token by concatenating the Snowflake-provided OAuth token for the service and the user token from this header, joined by `.`: `<service-token>.<caller-token>`. This combined token is then used to establish a connection to the rest of Snowflake using caller's rights. 

Check our our [documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/additional-considerations-services-jobs#connecting-to-snowflake-from-inside-a-container-using-caller-s-rights) for more!

### Conclusion
We're excited for these new capabilities and the ability go now build even more powerful and useful apps in SPCS. This capability is also [available for folks using Snowflake's Native Apps model](https://docs.snowflake.com/en/developer-guide/native-apps/restricted-callers-rights), currently in preview. Please let us know of any feedback in the comments. 