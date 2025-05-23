---
mapped_pages:
  - https://www.elastic.co/guide/en/machine-learning/current/ml-count-functions.html
products:
  - id: machine-learning
---

# Count functions [ml-count-functions]

Count functions detect anomalies when the number of events in a bucket is anomalous.

Use `non_zero_count` functions if your data is sparse and you want to ignore cases where the bucket count is zero.

Use `distinct_count` functions to determine when the number of distinct values in one field is unusual, as opposed to the total count.

Use high-sided functions if you want to monitor unusually high event rates. Use low-sided functions if you want to look at drops in event rate.

The {{ml-features}} include the following count functions:

* [`count`, `high_count`, `low_count`](ml-count-functions.md#ml-count)
* [`non_zero_count`, `high_non_zero_count`, `low_non_zero_count`](ml-count-functions.md#ml-nonzero-count)
* [`distinct_count`, `high_distinct_count`, `low_distinct_count`](ml-count-functions.md#ml-distinct-count)


## Count, high_count, low_count [ml-count]

The `count` function detects anomalies when the number of events in a bucket is anomalous.

The `high_count` function detects anomalies when the count of events in a bucket are unusually high.

The `low_count` function detects anomalies when the count of events in a bucket are unusually low.

These functions support the following properties:

* `by_field_name` (optional)
* `over_field_name` (optional)
* `partition_field_name` (optional)

For more information about those properties, see the [create {{anomaly-jobs}} API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-put-job).

```console
PUT _ml/anomaly_detectors/example1
{
  "analysis_config": {
    "detectors": [{
      "function" : "count"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

This example is probably the simplest possible analysis. It identifies time buckets during which the overall count of events is higher or lower than usual.

When you use this function in a detector in your {{anomaly-job}}, it models the event rate and detects when the event rate is unusual compared to its past behavior.

```console
PUT _ml/anomaly_detectors/example2
{
  "analysis_config": {
    "detectors": [{
      "function" : "high_count",
      "by_field_name" : "error_code",
      "over_field_name": "user"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

If you use this `high_count` function in a detector in your {{anomaly-job}}, it models the event rate for each error code. It detects users that generate an unusually high count of error codes compared to other users.

```console
PUT _ml/anomaly_detectors/example3
{
  "analysis_config": {
    "detectors": [{
      "function" : "low_count",
      "by_field_name" : "status_code"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

In this example, the function detects when the count of events for a status code is lower than usual.

When you use this function in a detector in your {{anomaly-job}}, it models the event rate for each status code and detects when a status code has an unusually low count compared to its past behavior.

```console
PUT _ml/anomaly_detectors/example4
{
  "analysis_config": {
    "summary_count_field_name" : "events_per_min",
    "detectors": [{
      "function" : "count"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

If you are analyzing an aggregated `events_per_min` field, do not use a sum function (for example, `sum(events_per_min)`). Instead, use the count function and the `summary_count_field_name` property. For more information, see [Aggregating data for faster performance](/explore-analyze/machine-learning/anomaly-detection/ml-configuring-aggregation.md).


## Non_zero_count, high_non_zero_count, low_non_zero_count [ml-nonzero-count]

The `non_zero_count` function detects anomalies when the number of events in a bucket is anomalous, but it ignores cases where the bucket count is zero. Use this function if you know your data is sparse or has gaps and the gaps are not important.

The `high_non_zero_count` function detects anomalies when the number of events in a bucket is unusually high and it ignores cases where the bucket count is zero.

The `low_non_zero_count` function detects anomalies when the number of events in a bucket is unusually low and it ignores cases where the bucket count is zero.

These functions support the following properties:

* `by_field_name` (optional)
* `partition_field_name` (optional)

For more information about those properties, see the [create {{anomaly-jobs}} API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-put-job).

For example, if you have the following number of events per bucket:

::::{admonition}
1,22,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,2,43,31,0,0,0,0,0,0,0,0,0,0,0,0,2,1

::::


The `non_zero_count` function models only the following data:

::::{admonition}
1,22,2,43,31,2,1

::::


```console
PUT _ml/anomaly_detectors/example5
{
  "analysis_config": {
    "detectors": [{
      "function" : "high_non_zero_count",
      "by_field_name" : "signaturename"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

If you use this `high_non_zero_count` function in a detector in your {{anomaly-job}}, it models the count of events for the `signaturename` field. It ignores any buckets where the count is zero and detects when a `signaturename` value has an unusually high count of events compared to its past behavior.

::::{note}
Population analysis (using an `over_field_name` property value) is not supported for the `non_zero_count`, `high_non_zero_count`, and `low_non_zero_count` functions. If you want to do population analysis and your data is sparse, use the `count` functions, which are optimized for that scenario.
::::



## Distinct_count, high_distinct_count, low_distinct_count [ml-distinct-count]

The `distinct_count` function detects anomalies where the number of distinct values in one field is unusual.

The `high_distinct_count` function detects unusually high numbers of distinct values in one field.

The `low_distinct_count` function detects unusually low numbers of distinct values in one field.

These functions support the following properties:

* `field_name` (required)
* `by_field_name` (optional)
* `over_field_name` (optional)
* `partition_field_name` (optional)

For more information about those properties, see the [create {{anomaly-jobs}} API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-put-job).

```console
PUT _ml/anomaly_detectors/example6
{
  "analysis_config": {
    "detectors": [{
      "function" : "distinct_count",
      "field_name" : "user"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

This `distinct_count` function detects when a system has an unusual number of logged in users. When you use this function in a detector in your {{anomaly-job}}, it models the distinct count of users. It also detects when the distinct number of users is unusual compared to the past.

```console
PUT _ml/anomaly_detectors/example7
{
  "analysis_config": {
    "detectors": [{
      "function" : "high_distinct_count",
      "field_name" : "dst_port",
      "over_field_name": "src_ip"
    }]
  },
  "data_description": {
    "time_field":"timestamp",
    "time_format": "epoch_ms"
  }
}
```

This example detects instances of port scanning. When you use this function in a detector in your {{anomaly-job}}, it models the distinct count of ports. It also detects the `src_ip` values that connect to an unusually high number of different `dst_ports` values compared to other `src_ip` values.

