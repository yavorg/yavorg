---
title: 'Snowflake Task-based orchestration for containerized Jobs'
date: 2026-03-12 10:00:00-08:00
featured_image: '/images/posts/spcs-jobs-tasks.png'
excerpt: Today we’re announcing improved Snowflake Task-based orchestration for containeriazed Jobs for ML Jobs and Snowpark Container Services customers
tags:
- Snowflake
---

Today we’re announcing **improved Snowflake Task-based orchestration for containeriazed Jobs (for ML Jobs and Snowpark Container Services customers)**. Tasks let users orchestrate Jobs natively inside Snowflake: scheduling or event-triggering containerized runs, chaining them into end-to-end pipelines, and keeping orchestration close to the data and the same governance and security model teams already rely on. 

With task graphs/DAGs customers can express multi-step dependencies (including parallel branches and fan-in/fan-out patterns) as a single coordinated workflow rather than stitching together external schedulers and custom glue code, and because execution is surfaced directly in Snowsight, operators get built-in observability — visual graph views, run history, node-level status, and fast drill-down straight into SPCS telemetry for troubleshooting and reruns. 

This feature set takes batch and ML/GPU-based workloads in Snowflake to the next level. Here are the specific improvements that underpin this new functionality:
  -  **Job execution in a serverless Task** - lets Snowflake orchestrate long-running Jobs using GS and Compute pools only, so a Warehouse never has to be resumed or kept running for the Job’s duration. This feature is key for compute-intensive workloads such as ML, where jobs could run for a long period of time on powerful GPU-based hardware, and price/performance is key for customers.
  - **Task context propagation in Jobs** - easy access to return values and graph configuration inside Job code via `get_predecessor_return_value()`, `set_return_value()`, and `get_task_graph_config()` methods (Python)
  - **ML Jobs with Tasks** - use an ML Job definition to define what a Task executes by passing in a definition, say a training function, into a `DAGTask()`
  - **Snowsight observability** - easy linking between Task runs / Query details  and Job details for improved debugging of failed Tasks and robust operational monitoring

![](/images/posts/spcs-jobs-tasks-screenshot.png)


Here is a short video demonstrating the new functionality: 
<iframe width="560" height="315" src="https://www.youtube.com/embed/fkLvNOHYXP8?si=0jmbu95n1k8Byau9" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Here are some additional resources to help you get started. 
* 📚 [SPCS documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/jobs-as-tasks)
* 🛠️ [SPCS step-by-step tutorial](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/tutorials/advanced/run-job-as-task)
* 🛠️ [ML Jobs with Tasks tutorial](https://github.com/Snowflake-Labs/sf-samples/tree/main/samples/ml/ml_jobs/e2e_task_graph)

We can't wait for you to try this out!