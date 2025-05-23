---
navigation_title: Run downsampling manually
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/downsampling-manual.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: elasticsearch
---



# Run downsampling manually [downsampling-manual]


The recommended way to [downsample](./downsampling-time-series-data-stream.md) a [time-series data stream (TSDS)](../data-streams/time-series-data-stream-tsds.md) is [through index lifecycle management (ILM)](run-downsampling-with-ilm.md). However, if you’re not using ILM, you can downsample a TSDS manually. This guide shows you how, using typical Kubernetes cluster monitoring data.

To test out manual downsampling, follow these steps:

1. Check the [prerequisites](#downsampling-manual-prereqs).
2. [Create a time series data stream](#downsampling-manual-create-index).
3. [Ingest time series data](#downsampling-manual-ingest-data).
4. [Downsample the TSDS](#downsampling-manual-run).
5. [View the results](#downsampling-manual-view-results).


## Prerequisites [downsampling-manual-prereqs]

* Refer to the [TSDS prerequisites](./set-up-tsds.md#tsds-prereqs).
* It is not possible to downsample a [data stream](../data-streams.md) directly, nor multiple indices at once. It’s only possible to downsample one time series index (TSDS backing index).
* In order to downsample an index, it needs to be read-only. For a TSDS write index, this means it needs to be rolled over and made read-only first.
* Downsampling uses UTC timestamps.
* Downsampling needs at least one metric field to exist in the time series index.


## Create a time series data stream [downsampling-manual-create-index]

First, you’ll create a TSDS. For simplicity, in the time series mapping all `time_series_metric` parameters are set to type `gauge`, but [other values](time-series-data-stream-tsds.md#time-series-metric) such as `counter` and `histogram` may also be used. The `time_series_metric` values determine the kind of statistical representations that are used during downsampling.

The index template includes a set of static [time series dimensions](time-series-data-stream-tsds.md#time-series-dimension): `host`, `namespace`, `node`, and `pod`. The time series dimensions are not changed by the downsampling process.

```console
PUT _index_template/my-data-stream-template
{
  "index_patterns": [
    "my-data-stream*"
  ],
  "data_stream": {},
  "template": {
    "settings": {
      "index": {
        "mode": "time_series",
        "routing_path": [
          "kubernetes.namespace",
          "kubernetes.host",
          "kubernetes.node",
          "kubernetes.pod"
        ],
        "number_of_replicas": 0,
        "number_of_shards": 2
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "kubernetes": {
          "properties": {
            "container": {
              "properties": {
                "cpu": {
                  "properties": {
                    "usage": {
                      "properties": {
                        "core": {
                          "properties": {
                            "ns": {
                              "type": "long"
                            }
                          }
                        },
                        "limit": {
                          "properties": {
                            "pct": {
                              "type": "float"
                            }
                          }
                        },
                        "nanocores": {
                          "type": "long",
                          "time_series_metric": "gauge"
                        },
                        "node": {
                          "properties": {
                            "pct": {
                              "type": "float"
                            }
                          }
                        }
                      }
                    }
                  }
                },
                "memory": {
                  "properties": {
                    "available": {
                      "properties": {
                        "bytes": {
                          "type": "long",
                          "time_series_metric": "gauge"
                        }
                      }
                    },
                    "majorpagefaults": {
                      "type": "long"
                    },
                    "pagefaults": {
                      "type": "long",
                      "time_series_metric": "gauge"
                    },
                    "rss": {
                      "properties": {
                        "bytes": {
                          "type": "long",
                          "time_series_metric": "gauge"
                        }
                      }
                    },
                    "usage": {
                      "properties": {
                        "bytes": {
                          "type": "long",
                          "time_series_metric": "gauge"
                        },
                        "limit": {
                          "properties": {
                            "pct": {
                              "type": "float"
                            }
                          }
                        },
                        "node": {
                          "properties": {
                            "pct": {
                              "type": "float"
                            }
                          }
                        }
                      }
                    },
                    "workingset": {
                      "properties": {
                        "bytes": {
                          "type": "long",
                          "time_series_metric": "gauge"
                        }
                      }
                    }
                  }
                },
                "name": {
                  "type": "keyword"
                },
                "start_time": {
                  "type": "date"
                }
              }
            },
            "host": {
              "type": "keyword",
              "time_series_dimension": true
            },
            "namespace": {
              "type": "keyword",
              "time_series_dimension": true
            },
            "node": {
              "type": "keyword",
              "time_series_dimension": true
            },
            "pod": {
              "type": "keyword",
              "time_series_dimension": true
            }
          }
        }
      }
    }
  }
}
```


## Ingest time series data [downsampling-manual-ingest-data]

Because time series data streams have been designed to [only accept recent data](time-series-data-stream-tsds.md#tsds-accepted-time-range), in this example, you’ll use an ingest pipeline to time-shift the data as it gets indexed. As a result, the indexed data will have an `@timestamp` from the last 15 minutes.

Create the pipeline with this request:

```console
PUT _ingest/pipeline/my-timestamp-pipeline
{
  "description": "Shifts the @timestamp to the last 15 minutes",
  "processors": [
    {
      "set": {
        "field": "ingest_time",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": """
          def delta = ChronoUnit.SECONDS.between(
            ZonedDateTime.parse("2022-06-21T15:49:00Z"),
            ZonedDateTime.parse(ctx["ingest_time"])
          );
          ctx["@timestamp"] = ZonedDateTime.parse(ctx["@timestamp"]).plus(delta,ChronoUnit.SECONDS).toString();
        """
      }
    }
  ]
}
```

Next, use a bulk API request to automatically create your TSDS and index a set of ten documents:

```console
PUT /my-data-stream/_bulk?refresh&pipeline=my-timestamp-pipeline
{"create": {}}
{"@timestamp":"2022-06-21T15:49:00Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":91153,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":463314616},"usage":{"bytes":307007078,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":585236},"rss":{"bytes":102728},"pagefaults":120901,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:45:50Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":124501,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":982546514},"usage":{"bytes":360035574,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":1339884},"rss":{"bytes":381174},"pagefaults":178473,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:44:50Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":38907,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":862723768},"usage":{"bytes":379572388,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":431227},"rss":{"bytes":386580},"pagefaults":233166,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:44:40Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":86706,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":567160996},"usage":{"bytes":103266017,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":1724908},"rss":{"bytes":105431},"pagefaults":233166,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:44:00Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":150069,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":639054643},"usage":{"bytes":265142477,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":1786511},"rss":{"bytes":189235},"pagefaults":138172,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:42:40Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":82260,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":854735585},"usage":{"bytes":309798052,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":924058},"rss":{"bytes":110838},"pagefaults":259073,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:42:10Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":153404,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":279586406},"usage":{"bytes":214904955,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":1047265},"rss":{"bytes":91914},"pagefaults":302252,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:40:20Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":125613,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":822782853},"usage":{"bytes":100475044,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":2109932},"rss":{"bytes":278446},"pagefaults":74843,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:40:10Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":100046,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":567160996},"usage":{"bytes":362826547,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":1986724},"rss":{"bytes":402801},"pagefaults":296495,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
{"create": {}}
{"@timestamp":"2022-06-21T15:38:30Z","kubernetes":{"host":"gke-apps-0","node":"gke-apps-0-0","pod":"gke-apps-0-0-0","container":{"cpu":{"usage":{"nanocores":40018,"core":{"ns":12828317850},"node":{"pct":2.77905e-05},"limit":{"pct":2.77905e-05}}},"memory":{"available":{"bytes":1062428344},"usage":{"bytes":265142477,"node":{"pct":0.01770037710617187},"limit":{"pct":9.923134671484496e-05}},"workingset":{"bytes":2294743},"rss":{"bytes":340623},"pagefaults":224530,"majorpagefaults":0},"start_time":"2021-03-30T07:59:06Z","name":"container-name-44"},"namespace":"namespace26"}}
```

You can use the search API to check if the documents have been indexed correctly:

```console
GET /my-data-stream/_search
```

Run the following aggregation on the data to calculate some interesting statistics:

```console
GET /my-data-stream/_search
{
    "size": 0,
    "aggs": {
        "tsid": {
            "terms": {
                "field": "_tsid"
            },
            "aggs": {
                "over_time": {
                    "date_histogram": {
                        "field": "@timestamp",
                        "fixed_interval": "1d"
                    },
                    "aggs": {
                        "min": {
                            "min": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        },
                        "max": {
                            "max": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        },
                        "avg": {
                            "avg": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        }
                    }
                }
            }
        }
    }
}
```


## Downsample the TSDS [downsampling-manual-run]

A TSDS can’t be downsampled directly. You need to downsample its backing indices instead. You can see the backing index for your data stream by running:

```console
GET /_data_stream/my-data-stream
```

This returns:

```console-result
{
  "data_streams": [
    {
      "name": "my-data-stream",
      "timestamp_field": {
        "name": "@timestamp"
      },
      "indices": [
        {
          "index_name": ".ds-my-data-stream-2023.07.26-000001", <1>
          "index_uuid": "ltOJGmqgTVm4T-Buoe7Acg",
          "prefer_ilm": true,
          "managed_by": "Unmanaged"
        }
      ],
      "generation": 1,
      "status": "GREEN",
      "next_generation_managed_by": "Unmanaged",
      "prefer_ilm": true,
      "template": "my-data-stream-template",
      "hidden": false,
      "system": false,
      "allow_custom_routing": false,
      "replicated": false,
      "rollover_on_write": false,
      "time_series": {
        "temporal_ranges": [
          {
            "start": "2023-07-26T09:26:42.000Z",
            "end": "2023-07-26T13:26:42.000Z"
          }
        ]
      }
    }
  ]
}
```

1. The backing index for this data stream.


Before a backing index can be downsampled, the TSDS needs to be rolled over and the old index needs to be made read-only.

Roll over the TSDS using the [rollover API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-rollover):

```console
POST /my-data-stream/_rollover/
```

Copy the name of the `old_index` from the response. In the following steps, replace the index name with that of your `old_index`.

The old index needs to be set to read-only mode. Run the following request:

```console
PUT /.ds-my-data-stream-2023.07.26-000001/_block/write
```

Next, use the [downsample API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-downsample) to downsample the index, setting the time series interval to one hour:

```console
POST /.ds-my-data-stream-2023.07.26-000001/_downsample/.ds-my-data-stream-2023.07.26-000001-downsample
{
  "fixed_interval": "1h"
}
```

Now you can [modify the data stream](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-modify-data-stream), and replace the original index with the downsampled one:

```console
POST _data_stream/_modify
{
  "actions": [
    {
      "remove_backing_index": {
        "data_stream": "my-data-stream",
        "index": ".ds-my-data-stream-2023.07.26-000001"
      }
    },
    {
      "add_backing_index": {
        "data_stream": "my-data-stream",
        "index": ".ds-my-data-stream-2023.07.26-000001-downsample"
      }
    }
  ]
}
```

You can now delete the old backing index. But be aware this will delete the original data. Don’t delete the index if you may need the original data in the future.


## View the results [downsampling-manual-view-results]

Re-run the earlier search query (note that when querying downsampled indices there are [a few nuances to be aware of](./downsampling-time-series-data-stream.md#querying-downsampled-indices-notes)):

```console
GET /my-data-stream/_search
```

The TSDS with the new downsampled backing index contains just one document. For counters, this document would only have the last value. For gauges, the field type is now `aggregate_metric_double`. You see the `min`, `max`, `sum`, and `value_count` statistics based off of the original sampled metrics:

```console-result
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 4,
    "successful": 4,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": ".ds-my-data-stream-2023.07.26-000001-downsample",
        "_id": "0eL0wC_4-45SnTNFAAABiZHbD4A",
        "_score": 1,
        "_source": {
          "@timestamp": "2023-07-26T11:00:00.000Z",
          "_doc_count": 10,
          "ingest_time": "2023-07-26T11:26:42.715Z",
          "kubernetes": {
            "container": {
              "cpu": {
                "usage": {
                  "core": {
                    "ns": 12828317850
                  },
                  "limit": {
                    "pct": 0.0000277905
                  },
                  "nanocores": {
                    "min": 38907,
                    "max": 153404,
                    "sum": 992677,
                    "value_count": 10
                  },
                  "node": {
                    "pct": 0.0000277905
                  }
                }
              },
              "memory": {
                "available": {
                  "bytes": {
                    "min": 279586406,
                    "max": 1062428344,
                    "sum": 7101494721,
                    "value_count": 10
                  }
                },
                "majorpagefaults": 0,
                "pagefaults": {
                  "min": 74843,
                  "max": 302252,
                  "sum": 2061071,
                  "value_count": 10
                },
                "rss": {
                  "bytes": {
                    "min": 91914,
                    "max": 402801,
                    "sum": 2389770,
                    "value_count": 10
                  }
                },
                "usage": {
                  "bytes": {
                    "min": 100475044,
                    "max": 379572388,
                    "sum": 2668170609,
                    "value_count": 10
                  },
                  "limit": {
                    "pct": 0.00009923134
                  },
                  "node": {
                    "pct": 0.017700378
                  }
                },
                "workingset": {
                  "bytes": {
                    "min": 431227,
                    "max": 2294743,
                    "sum": 14230488,
                    "value_count": 10
                  }
                }
              },
              "name": "container-name-44",
              "start_time": "2021-03-30T07:59:06.000Z"
            },
            "host": "gke-apps-0",
            "namespace": "namespace26",
            "node": "gke-apps-0-0",
            "pod": "gke-apps-0-0-0"
          }
        }
      }
    ]
  }
}
```

Re-run the earlier aggregation. Even though the aggregation runs on the downsampled TSDS that only contains 1 document, it returns the same results as the earlier aggregation on the original TSDS.

```console
GET /my-data-stream/_search
{
    "size": 0,
    "aggs": {
        "tsid": {
            "terms": {
                "field": "_tsid"
            },
            "aggs": {
                "over_time": {
                    "date_histogram": {
                        "field": "@timestamp",
                        "fixed_interval": "1d"
                    },
                    "aggs": {
                        "min": {
                            "min": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        },
                        "max": {
                            "max": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        },
                        "avg": {
                            "avg": {
                                "field": "kubernetes.container.memory.usage.bytes"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

This example demonstrates how downsampling can dramatically reduce the number of documents stored for time series data, within whatever time boundaries you choose. It’s also possible to perform downsampling on already downsampled data, to further reduce storage and associated costs, as the time series data ages and the data resolution becomes less critical.

The recommended way to downsample a TSDS is with ILM. To learn more, try the [Run downsampling with ILM](./run-downsampling-with-ilm.md) example.

