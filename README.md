# Enhancing Data Validation with Great Expectations and Sending Logs to Azure Monitor
[Great Expectations](https://docs.greatexpectations.io/docs/) is a Python library that provides a framework for describing the acceptable state of data and then validating that the data meets those criteria.

We use Great Expectations for data validation, and export logs to Azure Log Analytics or Azure Monitor to generate data quality report, and use Data Docs which renders Expectations in a clean, human-readable format.
![image](https://github.com/cchenshu/Data-validation/assets/49240055/62ef3ad5-5e19-41ec-91e1-4e83f0bcaefc)

To install the great_expectations library for performing data validation, you can execute the following command using Python's package installer.
```sh
pip install great_expectations
```

When working with Great Expectations you use the following four core components to access, store, and manage underlying objects and processes: Data Context, Data Sources, Expectations, Checkpoints.
We use [NYC taxi data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) for this experiment.

## Configure Data Context 

The Data Context will provide you with access to a variety of utility and convenience methods. It is the entry point for using the Great Expectations Python APIs. 

In Great Expectations, a DataContext is a top-level object that is used to manage and organize all your expectations, validation results, and data assets. It provides a unified interface to the various parts of Great Expectations and allows you to easily interact with and manage your expectations and validations. 

Instantiate a data context using the following code:

```python
data_context_config = DataContextConfig(
    datasources={
        "transformed_data_source": DatasourceConfig(
            class_name="Datasource",
            execution_engine={"class_name": "PandasExecutionEngine"},
            data_connectors={
                "transformed_data_connector": {
                    "module_name": "great_expectations.datasource.data_connector",
                    "class_name": "RuntimeDataConnector",
                    "batch_identifiers": [
                        "environment",
                    ],
                }
            }
        )
    },
    store_backend_defaults=FilesystemStoreBackendDefaults(root_directory=work_path)
)
context = BaseDataContext(project_config=data_context_config)
```

## Create a Batch Request 

A Batch Request is always used when Great Expectations builds a Batch. Any time you interact with something that requires a Batch of Data (such as a Profiler, Checkpoint, or Validator) you will use a Batch Request to create the Batch that is used.

```python
batch_request = RuntimeBatchRequest(
    datasource_name="transformed_data_source",
    data_connector_name="transformed_data_connector",
    data_asset_name="sampledataaset",
    batch_identifiers={
        "environment": "stage",
    },
    runtime_parameters={"batch_data": pd_df},
)
```

## Define Expectation Suite and corresponding Data Expectations 

An Expectation Suite is a collection of verifiable assertions about data. We need to add Validations to the suite. 

A Validator is a robust object capable of storing Expectations about the data it is associated with, as well as performing introspections on that data. 

You can check available expectations using validator.list_available_expectation_types(). 

You can save an Expectation Suite by using a Validator's  save_expectation_suite() method.

```python
expectation_suite_name = "Nyctaxi_data_suite_basic"
context.add_or_update_expectation_suite(expectation_suite_name=expectation_suite_name)
validator = context.get_validator(
    batch_request=batch_request,
    expectation_suite_name=expectation_suite_name,
    datasource_name="transformed_data_source",
    data_connector_name="transformed_data_connector",
    data_asset_name="sampledataaset",
)
validator.expect_column_values_to_be_between(column="passenger_count", min_value=0, max_value=10)
validator.expect_column_values_to_not_be_null(column="passenger_count")
validator.expect_column_values_to_not_be_null(column="trip_distance")
validator.expect_column_values_to_be_of_type(column="store_and_fwd_flag", type_="object")
validator.expect_column_values_to_not_be_null(column="fare_amount")
validator.expect_column_values_to_be_of_type(column="payment_type", type_="int")
validator.save_expectation_suite(discard_failed_expectations=False)
```

## Validate Data 

Great Expectations recommends using Checkpoints to validate data. Checkpoints validate data, save Validation Results, run any Actions you have specified, and finally create Data Docs with their results. A Checkpoint can be reused to Validate data in the future, and you can create and configure additional Checkpoints for different business requirements. 


Now that we have defined our Expectations it is time for Great Expectations to introspect our data and see if it corresponds to what we told Great Expectations to expect. To do this, we define a Checkpoint (which will allow us to repeat the Validation in the future). 

```python
my_checkpoint_name = "Nyctaxi Data"
checkpoint_config = {
    "name": my_checkpoint_name,
    "config_version": 1.0,
    "class_name": "SimpleCheckpoint",
    "run_name_template": "%Y%m%d-%H%M%S-my-run-name-template",
}
my_checkpoint = context.test_yaml_config(yaml.dump(checkpoint_config,default_flow_style=False))
context.add_or_update_checkpoint(**checkpoint_config)
# Run Checkpoint passing in expectation suite
checkpoint_result = context.run_checkpoint(
    checkpoint_name=my_checkpoint_name,
    validations=[
        {
            "batch_request": batch_request,
            "expectation_suite_name": expectation_suite_name,
        }
    ],
)
```

## Data Quality Metric Reporting 

### Send logs to Log Analytics 

Report Data Quality Metrics to Azure Monitor using python Azure Monitor open-census exporter. This parses the results of the checkpoint and sends it to AppInsights / Azure Monitor for reporting. 

We send data asset names, data source names that we used for validation, and the results for each validation.  
![image](https://github.com/cchenshu/Data-validation/assets/49240055/bea9dd8f-c102-40d1-93eb-cf5a64c0b476)

When some validations fail, we also send the detailed failure information to Log Analytics for the future debugging. 
![image](https://github.com/cchenshu/Data-validation/assets/49240055/219bef9b-e51d-4d56-a382-172aa8f29201)

### Create an alert rule 

Create an alert rule to get people notified when the data validation process is finished or if some validations fail. 

When the alert is triggered, people in the action group will get notified by email, txt, voice or push. 

For creating the alert rule, we can follow this page: [Create or edit an alert rule](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-create-new-alert-rule?tabs=metric).

In this demo, we create a log alert, use the following query to check whether validation process succeeds.

```
traces
| extend level = parse_json(customDimensions).level
| where level == 'ERROR'
```

Then we can get an email notification if data validation fails.

### Generate Data Docs 

Great Expectations renders Expectations in a clean, human-readable format called Data Docs. These HTML docs contain both your Expectation Suites, and your data Validation Results each time validation is run – think of it as a continuously updated data quality report. The following image shows a sample Data Doc: 

![image](https://github.com/cchenshu/Data-validation/assets/49240055/26230b18-a4d0-4c1d-8da2-919ab6aecc95)

## Integrate data validation into Data pipeline 

In the end-to-end pipeline, we add a new activity `Data Validation` after ingesting the data. If `Data Validation` fails, the following activities like cleansing, tranformation, and etc will not be executed. 
