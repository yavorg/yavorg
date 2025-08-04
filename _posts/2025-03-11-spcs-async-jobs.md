---
title: 'Evolving Snowflake Jobs: asynchronous processing'
date: 2025-03-11 10:10:00-08:00
excerpt: As part of Snowpark Container Services (SPCS), Snowflake offers a Jobs concept, which is well-suited for some important workloads in AI/ML and Data Engineering. By using the optional asynchronous Job processing support, users can save time and reduce cost with long-running workloads. 
tags:
- Snowflake
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/YZ7DR_JDSQo?si=m8lGrwl6-TGjqaFO" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

As part of Snowpark Container Services (SPCS), Snowflake offers a Jobs concept, which is well-suited for some important workloads:
- **AI/ML**: model training, inference, feature engineering, classical ML 
- **Data Engineering**: custom ETL (reuse existing frameworks or code), data preparation, batch processing 

A Job enables you to run a containerized (Docker) workload on various CPU/GPU configurations, work with Snowflake data via SQL or block storage to process data, and shut down and release resources when the Job is done. 
> Over the next few quarters, we'll be investing in multiple usability improvements to make the Jobs model an even better fit for the above scenarios.

This week we will talk about asynchronous processing support in Jobs - the ability to run multiple jobs in parallel. 

### Asynchronous processing
[This tutorial](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/tutorials/tutorial-2) gives a broad overview of how to use Jobs for a simple task such as processing data in a table. The Job is executed synchronously by the [`execute job service` statement](https://docs.snowflake.com/en/sql-reference/sql/execute-job-service), meaning the SQL query will not terminate until the Job is complete. 

With longer-running Jobs, this starts to present some challenges:
- The SQL session is blocked while the Job executes, meaning in interactive use-cases, the user can't do anything else until the Job is done
- If the SQL session is cancelled or disconnects, the Job is cancelled, meaning the user has to maintain a connection for the whole Job duration
- The Snowflake Warehouse used to run the `execute` statement continues to run and incur charges while the Job is running, while not delivering value to the user, since the sesison is blocked. 

To motivate this further, let's introduce a ML customer scenario: doing *text analytics on table data*. The code for this sample is available [here](https://github.com/Snowflake-Labs/spcs-templates/tree/main/ml_experience/async_job). We have developed a container that supports a variety of text analysis tasks, including *summarization* and *sentiment analysis*. 

Here is an example of a summarization Job running over a table containing Google reviews. 

```sql
EXECUTE JOB SERVICE IN COMPUTE POOL CPU_S
  NAME=summarization_job_sync
  FROM SPECIFICATION $$
  spec:
      container:
      - name: main
        image: REGISTRY/REPO/TEXT_ANALYSIS_IMAGE:TEXT_ANALYSIS_IMAGE_VERSION
        env:
          SNOWFLAKE_WAREHOUSE: XSMALL
        args:
        - "--task=summarization"
        - "--source_table=google_reviews.google_reviews.sample_reviews"
        - "--source_id_column=REVIEW_HASH"
        - "--source_value_column=REVIEW_TEXT"
        - "--result_table=results"
 $$;
```

This task executes synchronously and completes in about 3 minutes, while processing 100 rows of data on a XS Compute Pool. 

Let's now modify this to use the [optional `async` property](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/working-with-services#execute-a-job-service) to the Job to trigger asynchronous execution. Infact, let's run two copies of the same Job, with one configured to do summarization, and the other to do sentiment analysis:

```sql
EXECUTE JOB SERVICE IN COMPUTE POOL CPU_S
  NAME=summarization_job_async
  ASYNC=TRUE
  FROM SPECIFICATION $$
  spec:
      container:
      - name: main
        image: REGISTRY/REPO/TEXT_ANALYSIS_IMAGE:TEXT_ANALYSIS_IMAGE_VERSION
        env:
          SNOWFLAKE_WAREHOUSE: XSMALL
        args:
        - "--task=summarization"
        - "--source_table=google_reviews.google_reviews.sample_reviews"
        - "--source_id_column=REVIEW_HASH"
        - "--source_value_column=REVIEW_TEXT"
        - "--result_table=results"
$$;

EXECUTE JOB SERVICE IN COMPUTE POOL CPU_S
  NAME=sentiment_job_async
  ASYNC=TRUE
  FROM SPECIFICATION $$
  spec:
      container:
      - name: main
        image: REGISTRY/REPO/TEXT_ANALYSIS_IMAGE:TEXT_ANALYSIS_IMAGE_VERSION
        env:
          SNOWFLAKE_WAREHOUSE: XSMALL
        args:
        - "--task=sentiment"
        - "--source_table=google_reviews.google_reviews.sample_reviews"
        - "--source_id_column=REVIEW_HASH"
        - "--source_value_column=REVIEW_TEXT"
        - "--result_table=results"
$$;
```

The two Jobs will now run asynchronously in parallel, with each one spinning up it's own instance of the container to process its data. The above query now completes in about 11 seconds, and the Jobs complete asynchronously in the background.  

### Orchestrating Jobs within a SQL workflow

Now that we are able to run Jobs in parallel, we need to consider how to orchestrate SQL workflows around those Jobs. For example, you may wish to run some DML queries to work with the results of parallel Jobs. To wait for the completion of _all_ Jobs, SPCS provides the [`job!spcs_wait_for()`](https://docs.snowflake.com/en/sql-reference/functions/spcs_wait_for) function, which can be invoked in the above context as follows:

```sql
CALL summarization_job_async!spcs_wait_for('DONE', 600);
CALL sentiment_job_async!spcs_wait_for('DONE', 600);
```

The `spcs_wait_for` function takes as first argument the Job status to wait for, and the maximum timeout (in seconds) to wait for. To wait for multiple Jobs, just invoke the function sequentially, once for each Job in the parallel set. Using this function to wait for a single Job works, however at that point, you might as well just use a synchronous Job. The power here is scaling out your workload horizontally by running many concurrent Jobs, without using the ability to detect the overall completion of the work. 

### Reviewing Job execution history

Once a Job executes, it's the metadata about the execution is retained for 7 days. This is valuable if you kick off long-running Jobs and may need to check back on the outcome of a run on a subsequent day. You can access the execution history via the [`get_job_history()`](https://docs.snowflake.com/en/sql-reference/functions/get_job_history) table function as shown:

```sql
SELECT 
  name,
  status,
  created_time,
  completed_time,
  DATEDIFF('second', created_time, completed_time) AS duration_seconds
FROM TABLE(SNOWFLAKE.SPCS.GET_JOB_HISTORY(RESULT_LIMIT=>5));
```

See the documentation linked above for the full list of fields available as part of the history. 

### Reviewing Job logs

Pairing with the history view above, you likely also want to access the logs generated by the Job after the job completes. This is especially useful for troubleshooting a Job that may have failed unexpectedly. By default, all logs emitted to `stdout` and `stderr` by the container are stored in your Snowflake [account-level Event Table](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/monitoring-services#using-event-table) and can easily be retrieved by querying the table directly (as shown at the link) or by using the [`job!spcs_get_logs()`](https://docs.snowflake.com/en/sql-reference/functions/spcs_get_logs) convenience function like so:


```sql
SELECT * FROM TABLE(summarization_job_sync!spcs_get_logs());
```


### Next steps
Over the next few quarters, we will be detailing further improvements to Jobs such as batch processing, execution history, and scheduling, among others. 

 The code for this sample is available [here](https://github.com/Snowflake-Labs/spcs-templates/tree/main/ml_experience/async_job) if you'd like to try it out. 