_**LAST UPDATED:** 2/6/2024, by [Ran Yahalom](https://wix.slack.com/archives/D028P8YJY64)_

<!-- TOC -->
* [What is an offline model?](#what-is-an-offline-model)
* [How does batch prediction work?](#how-does-batch-prediction-work)
* [Pre-configuration tests](#pre-configuration-tests)
* [Setting up a Batch Config via the ML platform UI](#setting-up-a-batch-config-via-the-ml-platform-ui)
* [Triggering a batch prediction model](#triggering-a-batch-prediction-model)
* [Batch prediction job statuses](#batch-prediction-job-statuses)
* [Batch prediction requirements and limitations](#batch-prediction-requirements-and-limitations)
* [Simplifying batch prediction using the `wixml_extension` library](#simplifying-batch-prediction-using-the-wixml_extension-library)
* [Monitoring a batch prediction job](#monitoring-a-batch-prediction-job)
  * [Batch Job Details screen](#batch-job-details-screen)
  * [Model Log](#model-log)
  * [BI events](#bi-events)
* [Troubleshooting](#troubleshooting)
* [When should I prefer triggering my offline model directly instead of batch prediction via ML platform?](#when-should-i-prefer-triggering-my-offline-model-directly-instead-of-batch-prediction-via-ml-platform)
* [How can I pass offline model predictions as input to another online model?](#how-can-i-pass-offline-model-predictions-as-input-to-another-online-model)
* [Assignment](#assignment)
<!-- TOC -->

# What is an offline model?
👉 An _offline model_ is a model that doesn't need to be invoked in real-time. 

👉 Offline models are often invoked periodically with the help of a scheduling framework such [Apache Airflow](https://airflow.apache.org/). 

👉 Currently, the only way to trigger an offline model through ML platform is by configuring it for **"Batch Prediction"**.

👉 In contrast to online models, batch prediction models can be efficiently invoked on relatively large datasets by splitting them into smaller batches on which the model is invoked simultaneously. For a summarized comparison between online and batch prediction models, please review the [Introduction lesson](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/00_Introduction_to_ML_Platform/Introduction.md). The two major benefits of batch prediction compared to online models are:
   1. **Performance on large datasets** (i.e. large number of rows) is much faster.
   2. **Cost-saving**. Batch prediction initializes the resources it needs on request (AWS Sagemaker instances) and terminates them once the operation is done (unlike the online models where Sagemaker resources are kept alive 24/7).

👉 If your offline model doesn't need to be invoked on large datasets that require batch prediction, you should consider triggering it directly from your code rather than using batch prediction (more on this use-case below).  

# How does batch prediction work?
👉 The following diagram illustrates the most common two stages of batch prediction - a one-time setup, and a recurring scheduled batch prediction flow:
![how_does_batch_prediction_work.png](ml-platform-course/05_Offline_Models/how_does_batch_prediction_work.png)

⚠️ NOTE: Learning to automate scheduled tasks with Airflow (step 4 of stage 1 and steps 1,2,3 and 11 of stage 2) is out of scope for this lesson. Similarly, you do not need to know the bits and bytes of the ML platform architecture that implements a batch prediction job (step 4 of stage 2). Nonetheless, you should be at least broadly familiar with these concepts as they will clarify different aspects of your work with batch prediction (e.g. configuration or troubleshooting).   

👉 In the first stage, you set up batch prediction in the following steps:
   1. First you should deploy & trigger the relevant model build as an online model to verify that all its "non-batch prediction related" functionality works on ML platform.    
   2. Next, you configure your model for batch prediction with the same settings (e.g. machine type) that worked for you in the previous step.  
   3. Now you should trigger a batch prediction job, either programmatically from your local computer or through the ML platform UI, to verify that everything works.
   4. Finally, you can use Airflow to create and run a directed acyclic graph (DAG) representation of the reoccurring scheduled sequence of tasks that make up the batch prediction flow.  

👉 Once your DAG is up and running, the following sequence of steps will be repeatedly executed at scheduled intervals:
   1. In most cases, your DAG will need to carry out different tasks before populating the input table with the data for prediction. For example, you may need to preprocess your input data before saving it to the input table.
   2. The next task is to populate the input table with the prediction data.
   3. Next, your DAG will typically invoke the [`BatchPredictor`](https://github.com/wix-private/data-science/blob/master/ml-framework/wixml-python-clients/wixml_clients/batchpredictor.py#L84) class from the `wixml_clients` library on your batch configured model.
   4. The [`BatchPredictor`](https://github.com/wix-private/data-science/blob/master/ml-framework/wixml-python-clients/wixml_clients/batchpredictor.py#L84) invocation will cause the ML platform to initiate a new batch prediction job:
      - This job is orchestrated by a dedicated Airflow DAG, not to be confused with your DAG, which in turn triggers an [AWS Sagemaker Batch Transform](https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html) job on the data from your input table. 
      - Sagemaker will split the input data into batches (i.e. having the same columns as the input data but with a subset of the rows), then simultaneously invoke your model's `predict()` method on each batch and return its output dataframe joined with the input dataframe. 
      - You can specify the type of Sagemaker machine instances on which the batches will be processed when you set up the batch configuration in the first stage. However, you have no control over the number of batches/instances; this is determined by ML platform automatically according to the size of the input. 
      - After the processing completes for all batches, Sagemaker will concatenate the results into a single dataframe and return it to the batch prediction job DAG, which will then write it to the output table and finish the batch prediction job.
   5. Finally, your DAG can execute some post-processing tasks on your output table, e.g. copy your model's prediction results to a permanent log table, so it won't be lost on the next batch prediction. 

# Pre-configuration tests
👉 Before setting up a Batch Config for your offline model, you should test it as follows:
   1. Deploy the appropriate build of your model as an online model. This will allow you to verify that the machine type you choose has enough resources to load your model.
   2. Invoke the deployed online model using the wixml_clients library on a small prediction data frame. This will allow you to verify that your model's schema is correct.
👉 You can review the [**"Online models"**](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/04_Online_Models/Online_models.md) lesson to recall how to [deploy](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/04_Online_Models/Online_models.md#deploying-an-online-model-manually-via-the-ml-platform-ui) & [trigger](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/04_Online_Models/Online_models.md#directly-triggering-an-online-model) online models.

# Setting up a Batch Config via the ML platform UI
👉 To configure your offline model for batch prediction via the ML platform UI, follow these steps:
1. Go to the BUILDS tab in your model's overview page, find the successful build you wish to configure for batch prediction in the list and click its vertical ellipsis ("⋮"):
   ![img.png](model_deployment.png)

2. Click "Set Batch Config" to open the "Batch Config Build" modal:
   ![img.png](batch_config_modal.png)
   Fill in the following fields:
    - Deployment Name: choose a name that will help you to identify your batch configuration.
    - Instance type: the machine type you chose to use in the pre-configuration test step.
    - Click on the "Show Advanced Configuration" button to change the default number of processes or define environment variables: 
      - Number of Processes: if you are using TensorFlow you must select ONE. As a general rule of thumb - if your model consumes a lot of memory (e.g. more than 1GB of memory), we recommend selecting ONE, otherwise select ALL.
      - Environment variables: key-value pairs that will be set as environment variables during model runs so that you can use in your model code. Only CAPS and underscore characters are allowed. An example can be a variable defining the debug mode which your code can use to print out some additional information only when debugging.

👉 After you click the "Deploy" button, the configuration you created will be set as the "active batch configuration" for the build you chose and the "Used in Batch" indicator will be shown:
   ![img.png](used_in_batch_indicator.png)

⚠️ NOTE: Setting up a batch configuration only saves the configuration you provided, so that it can later be used when you trigger a batch prediction process. It does not deploy or run your model!!!

# Triggering a batch prediction model
👉 Once your model has an active batch configuration, you can trigger a batch prediction job by providing the names of your input and output tables in the PREDICT tab of your model's overview page, and clicking on the "Trigger Batch Job" button:
   ![img.png](trigger_batch_prediction_via_ui.png)

👉 You can also trigger a batch prediction job using the `wixml_clients` libray (version >= 1.0.9) as described [here](https://wix-data-science.wixanswers.com/kb/en/article/how-to-use-batch-prediction-transformation#batch-prediction-triggering-via-wixml) or simply using the [`invoke_batch_predict_model()`](https://github.com/wix-private/ds-general/blob/860cec031b46cd69f4a9d0659f380358370784fa/wixml-extensions/wixml_extensions/mlflow.py#L150) helper function from the `wixml_extensions` package as follows:
```python
from wixml_extensions.mlflow import invoke_batch_predict_model

predictor_config, batch_predictor = invoke_batch_predict_model(model_id='ds-premium-fraud-detection',
                                                               input_table_name='prod.fraud_in_premium_model.input',
                                                               output_table_name='prod.fraud_in_premium_model.output')
```

# Batch prediction job statuses
👉 Once a job is finished it may have one of the following statuses:
   - **Success**: indicates the process succeeded from an infrastructure point of view. This means that the process created a Sagemaker endpoint, loaded the input table, executed inference, collected results, and wrote into the output table.
   - **Failure**: indicates the process could not perform all required steps. It may be as a result of a bad user request such as passing an output table that already exists, or an infrastructure failure such as Sagemaker endpoint creation failure.

⚠️ NOTE: A success status applies even if only a single model invocation succeeded and the model returned errors for all other prediction requests! It is up to you to check that the output table has the expected row count.

# Batch prediction requirements and limitations
[//]: # (TODO 3-6-24: Add description of what happens when a column in the output schema is missing from the output data frame returned by the predict method. Will it be added by ML platform as an empty column or will this result in an error?)
👉 To work with batch prediction you have to make sure the following requirements are satisfied or your batch prediction jobs will fail:
   1. Your model's `predict()` method must:
      1. Return a dataframe of the same size as the input and with the same order of rows.
      2. Complete its invocation in less than 1 hour.
   2. Your model's `schema()` method must:
      1. Return an output schema which includes a field for any column of the dataframe returned by the `predict()` method that you want the ML platform to collect and store in the output table. If a column of the dataframe returned by the `predict()` method is not included as a field in the output schema, it will not appear in your output table.
      2. The name fields of all [`Feature`](https://github.com/wix-private/data-services/blob/a4576da20db8010933fae088b95236e799f6b606/ml-framework/wix-python-ml/wixml/model/schema.py#L19) and [`Prediction`](https://github.com/wix-private/data-services/blob/a4576da20db8010933fae088b95236e799f6b606/ml-framework/wix-python-ml/wixml/model/schema.py#L67) objects in the returned [`ModelSchema`](https://github.com/wix-private/data-services/blob/a4576da20db8010933fae088b95236e799f6b606/ml-framework/wix-python-ml/wixml/model/schema.py#L125) must contain lowercase strings.
   3. The input table:
      1. Doesn't contain more than the allowed maximum number of rows, which is currently 5M. 
      2. Complies with the required input schema. The input table can contain additional columns, but if column _`x`_ appears in the model input schema - then column _`x`_ must also appear in your input table.
       
          ⚠️ NOTE: Following recent changes, the input table [data types](https://prestodb.io/docs/current/language/types.html) are no longer limited to primitives (i.e. `varchar`, `boolean`, `bigint`, or `real`) and ML platform ignores the `type` field of the input schema [`Feature`](https://github.com/wix-private/data-services/blob/a4576da20db8010933fae088b95236e799f6b606/ml-framework/wix-python-ml/wixml/model/schema.py#L19) class. 
      3. Doesn't contain a column that is returned from the prediction of the model. That's because the final output of batch prediction is a joined table of the input table (from the user) and the prediction. In case there is a mutual column _`x`_, the job won't know which one to take - _`x`_ from the input table or _`x`_ from the prediction.
      4. Doesn't contain rows that are larger than 100 MB in size when converted to bytes. You should take this into consideration when your data includes very large blobs / text.
   4. The output table doesn't exist.
   5. The overall batch prediction process should not take more than 8 hours.

👉 You should also be aware of the following usage limitation: since every batch prediction job initiates new AWS Sagemaker batch transform job instances, triggering too many requests at the same time should be avoided because it may cause resource starvation. Until there is a proper queueing mechanism for it, you should **avoid triggering more than a single batch prediction job at a time**.

# Simplifying batch prediction using the [`wixml_extension`](https://github.com/wix-private/ds-general/tree/master/wixml-extensions#batch-prediction) library
👉 If your model's class extends the [`BatchPredictTransformationModel`](https://github.com/wix-private/ds-general/blob/37d98b375cc74a56d3eab5c40fbca3c10be41501/wixml-extensions/wixml_extensions/models.py#L960) class included in the [`wixml_extension`](https://github.com/wix-private/ds-general/tree/master/wixml-extensions#batch-prediction) library:
   1. The [`BatchPredictTransformationModel`](https://github.com/wix-private/ds-general/blob/37d98b375cc74a56d3eab5c40fbca3c10be41501/wixml-extensions/wixml_extensions/models.py#L960) will verify that your model's implementation satisfies all the necessary [batch prediction requirements](#batch-prediction-requirements-and-limitations) by the time you run your model’s predict() method locally on your computer. This can significantly decrease the time & effort to fix all problems before you ever build your model on ML Platform (compared to repeated iterations of build->deploy->invoke on the ML Platform for each problem).
   2. All you have to do to invoke a complete batch prediction flow (including setting up the input table, deleting a pre-existing output table and calling the batch prediction client) is to run two lines of code, as demonstrated by the following snippet:
 ```python
from wixml_extensions.models import BatchPredictTransformationModel

def run_batch_predict_flow(model_instance: BatchPredictTransformationModel):
      model_instance.prepare_for_batch_predict()
      model_instance.invoke_batch_predict()
 ```
👉 Note that the `run_batch_predict_flow()` function in the above snippet internally calls the previously mentioned `wixml_extensions.mlflow.invoke_batch_predict_model()` function which has an additional benefit. After the batch prediction job completes, the function will verify for you that all batches completed successfully by checking if the output table has the same number of rows as the input table. If not, the function will raise an informative exception to make you aware of this.  

👉 You are encouraged to read all about extending the [`BatchPredictTransformationModel`](https://github.com/wix-private/ds-general/blob/37d98b375cc74a56d3eab5c40fbca3c10be41501/wixml-extensions/wixml_extensions/models.py#L960) [here](https://github.com/wix-private/ds-general/tree/master/wixml-extensions#batch-prediction) and contact [Ran Yahalom](https://wix.slack.com/archives/D028P8YJY64) if you have any questions.

# Monitoring a batch prediction job
## Batch Job Details screen
👉 Batch prediction jobs are exposed via the ML platform UI. You can easily see the details for your batch job by clicking on your batch job in the [Batch Jobs Board](https://wix-data-science.wixanswers.com/en/article/batch-jobs-overview-wip) page:
   ![bath_job_details.png](/ml-platform-course/05_Offline_Models/batch_job_details.png)

## Model Log
👉 If your model prints messages to the standard output, you can inspect them in the batch job's CloudWatch log. You can find this log as follows:
   1. Login to AWS by going to https://sso.wewix.net/auth/realms/WIX-APP/protocol/saml/clients/kgb-aws and signing in with the **`kgb-aws-415208756561-Data-Science-ReadOnly`** role (more details on this [here](https://github.com/wix-private/aws-sso#login-into-aws)).
   2. Go to [this](https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups/log-group/$252Faws$252Fsagemaker$252FTransformJobs) CloudWatch log group page.
   3. Search for your `job_id` in the log streams search box. You can find your `job_id` as described in the [Troubleshooting](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/05_Offline_Models/Batch_prediction_models.md#troubleshooting) section below.

## BI events
👉 The platform sends the following (debug level) BI events, available within a 15-minute delay in `sites_97` table:
   - Batch prediction job start - [97:402](https://bo.wix.com/bi-catalog-webapp/#/sources/97/events/402?artifactId=com.wixpress.bi.batch-predictor-web)
   - Batch-Predictor Job Success - [97:400](https://bo.wix.com/bi-catalog-webapp/#/sources/97/events/400?artifactId=com.wixpress.bi.batch-predictor-web)
   - Batch-Predictor Job Failed - [97:401](https://bo.wix.com/bi-catalog-webapp/#/sources/97/events/401?artifactId=com.wixpress.bi.batch-predictor-web)

# Troubleshooting
👉 To troubleshoot a failed batch prediction job, you need to find its _job ID_ in any of the following:
   - Batch prediction Airflow DAG log:
   ![img.png](find_failed_batcprediction_job_id_from_airflow_log.png)
   - Local dev environment console output:
   ![img.png](find_failed_batcprediction_job_id_from_wixml_client_output.png)
   - The [Batch Jobs Board](https://bo.wix.com/ml-platform/batch) in the ML platform UI:
   ![img.png](find_failed_batcprediction_job_id_from_ml_platform_ui.png)
   - Using the [batch prediction job start](https://bo.wix.com/bi-catalog-webapp/#/sources/97/events/402?artifactId=com.wixpress.bi.batch-predictor-web) BI event (keeping in mind there is a 15 min. delay for events to be available via Quix):
```sql
select 
 src
    , evid
    , uuid
    , build_id
    , endpoint_name
    , job_id
    , job_start_time
    , job_type
    , model_id
    , date_created
from events.dbo.sites_97
where
    evid = 402 and
    model_id = 'ds-premium-fraud-detection'
order by date_created desc
```
![img.png](find_failed_batcprediction_job_id_from_bi_event.png)

👉 Next, find & click your job ID in the [Batch Jobs Board](https://bo.wix.com/ml-platform/batch) of the ML platform UI. Click the "Show More" in the red "Batch Job Message" panel to see the error message and a link to the relevant CloudWatch logs where you can further inspect messages logged on the Sagemaker instances, including your own code's messages:
![img.png](faild_batch_prediction_job_cloud_watch_logs.png)

   ⚠️ NOTE: you need to be logged into AWS in order to access the CloudWatch logs. Do this as described in the [Model Log](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/05_Offline_Models/Batch_prediction_models.md#model-log) section above.  

👉 You can also inspect the error that caused the failure by fetching it from the associated _[Batch-Predictor Job Failed](https://bo.wix.com/bi-catalog-webapp/#/sources/97/events/401?artifactId=com.wixpress.bi.batch-predictor-web)_ BI event using an SQL query like this:
```sql
select error
from events.dbo.sites_97
where job_id = 'f2d2f9cf-2a72-4abe-ae74-f962a7b70a51'
and error is not null
```
  ![img.png](inspect_batch_predictor_job_failed_bi_event.png)

# When should I prefer triggering my offline model directly instead of batch prediction via ML platform?
👉 If you want to invoke your offline model periodically on a relatively **SMALL** dataset which doesn't require splitting into smaller batches, you should consider instantiating your model and calling its `predict()` method directly from your code instead of using batch prediction.

👉 This has the following advantages:
1. You may achieve faster performance by saving some of the [previously discussed](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/00_Introduction_to_ML_Platform/Introduction.md#trigger-invoke-your-model) batch prediction overhead, e.g.:
    - The 10-15 minutes invocation latency incurred by batch prediction.
    - Mandatory batch prediction post-invocation steps such as writing results to an output DB table.
2. It saves you the administrative overhead of creating and managing batch configs.
3. The invocation flow is much simpler - there are no additional Airflow DAG & Sagemaker involved as in the case of batch prediction and there are overall less things that can go wrong.

👉 The main downside to this approach is that you don't have the convenient deployment configuration (e.g. machine type) or performance monitoring that you can get when using batch prediction.

👉 Using Airflow to schedule periodic triggering of an offline model is outside the scope of this lesson, but you get the idea from this [example](https://wix-data-science.wixanswers.com/kb/en/article/airflow-ds-use-cases). You are always more than welcome to consult with the [Payments DS team](https://app.slack.com/client/T02T01M9Y/C037GUSM7K5), that employs this approach extensively.

# How can I pass offline model predictions as input to another online model?
👉 The offline data needs to be stored as "synthetic event" (source 70) so it can be consumed online - like the site classification results.

[//]: # (TODO 3-6-24: Need to add link to the Synthetic Events Logger DAG mentioned below)
👉 You need to use the Synthetic Events Logger DAG to create synthetic events for your data (basically insert your data into a table).

# Assignment
👉 Conduct the first three steps of the batch prediction SETUP stage described [above](#how-does-batch-prediction-work). Don't forget to create an appropriate DB input table for your model before triggering batch prediction in the third step.

👏 Congratulations!!! You are now ready to continue to the next lesson and learn how to create and manage a research machine. 
