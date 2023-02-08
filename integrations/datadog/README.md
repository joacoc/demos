# Materialize + Datadog

<img width="1728" alt="Screenshot 2023-02-01 at 12 53 41" src="https://user-images.githubusercontent.com/11491779/216036715-9a4b4db7-8f93-4b6a-ac21-f7eb5a01d151.png">

## Overview

This is a demo of how to use Datadog to monitor Materialize using the OpenMetrics standard.

Materialize collects information about its internal components in a [public catalog](https://materialize.com/docs/sql/system-catalog/mz_internal/), consumable by tools like [Prometheus SQL Exporter](https://github.com/justwatchcom/sql_exporter/), _"a service that runs user-defined SQL queries at flexible intervals and exports the resulting metrics via HTTP for Prometheus consumption"_.

Datadog's OpenMetrics integration can consume from the Prometheus SQL Exporter endpoint and allows a way to monitor Materialize.

<img width="1499" alt="Datadog" src="https://user-images.githubusercontent.com/11491779/216926716-cc0cdd46-7ea7-496f-8fa3-a4863fc1ce00.png#gh-dark-mode-only">
<img width="1488" alt="Datadog" src="https://user-images.githubusercontent.com/11491779/217254889-adee0203-dd9c-4756-9c61-ee601c3f8e32.png#gh-light-mode-only">

## Prerequisites

- [Docker Compose](https://docs.docker.com/compose/install/)
- [Materialize instance](https://materialize.com/cloud/)
- [Datadog API Key](https://docs.datadoghq.com/account_management/api-app-keys/)

## Running the demo

1. Start by cloning the `demos` repository and navigating to the `datadog` directory:

    ```bash
    git clone https://github.com/MaterializeInc/demos.git
    cd integrations/datadog
    ```

2. Edit the `config.yaml` file and set your Materialize details under the `connections` key:

    ```yaml
    connections:
    - "postgres://<USER>:<PASSWORD>@<HOST>:<PORT>/materialize"
    ```


3. Edit the `docker-compose.yaml` file and set your Datadog API key:

    ```yaml
    environment:
    - DD_API_KEY=${DD_API_KEY}
    ```

4. Start the demo:

    ```bash
    docker-compose up
    ```

5. Open your Datadog account and explore the `materialize.*` metrics.

## Configuration overview

The `config.yaml` file contains the configuration for the Prometheus SQL exporter. Each job can have multiple configuration options, including:

1. The `connections` section where you can configure the Prometheus SQL exporter to connect to multiple Materialize instances:

```yaml
connections:
- "postgres://<USER>:<PASSWORD>@<HOST>:<PORT>/materialize"
```

You can change the interval at which the exporter queries Materialize by changing the `interval` value. The `connections` section contains the connection string for the Materialize instance.

2. The `queries` section where you can configure the Prometheus SQL exporter to export metrics from Materialize:

```yaml
queries:
- name: "replica_memory_usage"
    help: "Replica memory usage"
    labels:
    - "replica_name"
    - "cluster_id"
    values:
    - "memory_percent"
    query:  |
            SELECT
            name::text as replica_name,
            cluster_id::text as cluster_id,
            memory_percent::float as memory_percent
            FROM mz_cluster_replicas r join mz_internal.mz_cluster_replica_utilization u on r.id=u.replica_id;
```

The `queries` section contains all the queries that the Prometheus SQL exporter will run to export metrics from Materialize. For each query, you can define the following properties:
- **Name**: This is the name of the metric that will be exported to Prometheus
- **Labels**: These are the columns that will be used as labels in the exported metric
- **Values**: These are the columns that will be used as values in the exported metric
- **Query**: This is the SQL query that will be run to get the data for the metric

### Adding extra Prometheus SQL exporter checks

To configure the Prometheus SQL exporter to export additional metrics, add a new entry to the `queries` section of the `config.yaml` file. For example, to export the number of rows in the `orders` table, add the following:

```yaml
queries:
- name: "total_orders"
    help: "Total Orders"
    values:
    - "count"
    query:  |
            select count(*) from orders
```

## Datadog's OpenMetrics integration

Datadog's agent will pick up the `/conf.d/openmetrics.yaml` configuration file to consume the metrics available in the Prometheus SQL Exporter endpoint.


## Sources monitoring and alerting (Optional)

Sources can fail or become stale in diverse circumstances, due to connection issues or schema modifications. Most of the time it happens unintentionally and goes unnoticed for hours. Let's build a bolder operation over sources by monitoring and alerting using Datadog.

The idea is that when a source has a `stalled` or `failed` status, Datadog will respectively send an email warning or alert:

<details>
<summary>Steps</summary>

  1. Add a new query to the `config.yaml` to retrieve sources' statuses
  ```yaml
    - name: "source_statuses"
      help: "Source's statuses"
      values:
        - "status"
      labels:
        - "name"
        - "type"
      query:  |
              SELECT
                name,
                type,
                case
                  when status = 'failed' then 2
                  when status = 'stalled' then 1
                  else 0
                end as status
              FROM mz_internal.mz_source_statuses;
  ```
  2. Re-run `docker-compose` to load the new config.
  ```bash
      docker-compose up
  ```
  3. In Datadog, create a new [Metric Monitor](https://docs.datadoghq.com/monitors/types/metric/?tab=threshold).
     1. Choose **Threshold Alert** as the detection method.
     2. Define a unique metric as follows:
        1. Query `materialize.sql_source_statuses`
        2. From: `col:status`
        3. Avg by: `type` and `name`
     3. For the alert conditions:
        1. Pick `above or equal to` for measuring the threshold.
        2. Set the **Alert threshold** to `>= 2`
        3. Set the **Warning threshold** to `>= 1`
     4. Edit the alert message:
        ```
        {{#is_alert}}
            Alert: The source is failing. Check the status in Materialize.
        {{/is_alert}}

        {{#is_warning}}
            Warning: The source is stalled. Check the status in Materialize.
        {{/is_warning}} @your@email.com
        ```
     5. Click on **Test notifications** and assert the notification in your email.
     6. Click **Create**.

</details>

## Helpful links

- [Materialize](https://materialize.com/)
- [Materialize internal schema](https://materialize.com/docs/sql/system-catalog/mz_internal/)
- [Prometheus SQL Exporter](https://github.com/justwatchcom/sql_exporter/)
- [Datadog](https://www.datadoghq.com/)
