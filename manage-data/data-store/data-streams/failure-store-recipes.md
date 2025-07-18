---  
mapped_pages:
- (8.19 docs)

applies_to:
  stack: ga 9.1
  serverless: ga

products:
- id: elasticsearch
- id: elastic-stack
- id: cloud-serverless
---

# Using failure stores to address ingestion issues [failure-store-examples]

When something goes wrong during ingestion it is often not an isolated event. Included for your convenience are some examples of how you can use the failure store to quickly respond to ingestion failures and get your indexing back on track.

## Troubleshooting nested ingest pipelines [failure-store-examples-nested-ingest-troubleshoot]

When a document fails in an ingest pipeline it can be difficult to figure out exactly what went wrong and where. When these failures are captured by the failure store during this part of the ingestion process, they will contain additional debugging information. Failed documents will note the type of processor and which pipeline was executing when the failure occurred. Failed documents will also contain a pipeline trace which keeps track of any nested pipeline calls that the document was in at time of failure.

To demonstrate this, we will follow a failed document through an unfamiliar data stream and ingest pipeline:
```console
POST my-datastream-ingest/_doc
{
    "@timestamp": "2025-04-21T00:00:00Z",
    "important": {
      "info": "The rain in Spain falls mainly on the plain"
    }
}
```

```console-result
{
  "_index": ".fs-my-datastream-ingest-2025.05.09-000001",
  "_id": "F3S3s5YBwrYNjPmayMr9",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 2,
  "_primary_term": 1,
  "failure_store": "used" <1>
}
```
1. The document was sent to the failure store.

Now we search the failure store to check the failure document to see what went wrong.
```console
GET my-datastream-ingest::failures/_search
```

```console-result
{
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
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
        "_index": ".fs-my-datastream-ingest-2025.05.09-000001",
        "_id": "F3S3s5YBwrYNjPmayMr9",
        "_score": 1,
        "_source": {
          "@timestamp": "2025-05-09T06:24:48.381Z",
          "document": {
            "index": "my-datastream-ingest",
            "source": { <1>
              "important": {
                "info": "The rain in Spain falls mainly on the plain" <2>
              },
              "@timestamp": "2025-04-21T00:00:00Z"
            }
          },
          "error": {
            "type": "illegal_argument_exception",
            "message": "field [info] not present as part of path [important.info]", <3>
            "stack_trace": """j.l.IllegalArgumentException: field [info] not present as part of path [important.info]
	at o.e.i.IngestDocument.getFieldValue(IngestDocument.java:202)
	at o.e.i.c.SetProcessor.execute(SetProcessor.java:86)
	... 19 more
""",
            "pipeline_trace": [ <4>
              "ingest-step-1",
              "ingest-step-2"
            ],
            "pipeline": "ingest-step-2", <5>
            "processor_type": "set" <6>
          }
        }
      }
    ]
  }
}
```
1. When an ingest pipeline fails, the document stored is what was originally sent to the cluster.
2. The important information that we failed to find was originally present in the document.
3. The info field was not present when the failure occurred.
4. The first pipeline called the second pipeline.
5. The document failed in the second pipeline.
6. It failed in the pipeline's set processor.

Despite not knowing the pipelines beforehand, we have some places to start looking. The `ingest-step-2` pipeline cannot find the `important.info` field despite it being present in the document that was sent to the cluster. If we pull that pipeline definition we find the following:

```console
GET _ingest/pipeline/ingest-step-2
```

```console-result
{
  "ingest-step-2": {
    "processors": [
      {
        "set": { <1>
          "field": "copy.info",
          "copy_from": "important.info" <2> 
        }
      }
    ]
  }
}
```
1. There is only one processor here.
2. This field was missing from the document at this point.

There is only a set processor in the `ingest-step-2` pipeline so this is likely not where the root problem is. Remembering the `pipeline_trace` field on the failure we find that `ingest-step-1` was the original pipeline called for this document. It is likely the data stream's default pipeline. Pulling its definition we find the following:

```console
GET _ingest/pipeline/ingest-step-1
```

```console-result
{
  "ingest-step-1": {
    "processors": [
      {
        "remove": {
          "field": "important.info" <1>
        }
      },
      {
        "pipeline": {
          "name": "ingest-step-2" <2>
        }
      }
    ]
  }
}
```
1. A remove processor that is incorrectly removing our important field.
2. The call to the second pipeline.

We find a remove processor in the first pipeline that is the root cause of the problem! The pipeline should be updated to not remove important data, or the downstream pipeline should be changed to not expect the important data to be always present.

## Troubleshooting complicated ingest pipelines [failure-store-examples-complicated-ingest-troubleshoot]

Ingest processors can be labeled with tags. These tags are user-provided information that names or describes the processor's purpose in the pipeline. When documents are redirected to the failure store due to a processor issue, they capture the tag from the processor in which the failure occurred, if it exists. Because of this behavior, it is a good practice to tag the processors in your pipeline so that the location of a failure can be identified quickly.

Here we have a needlessly complicated pipeline. It is made up of several set and remove processors. Beneficially, they are all tagged with descriptive names.
```console
PUT _ingest/pipeline/complicated-processor
{
  "processors": [
    {
      "set": {
        "tag": "initialize counter",
        "field": "counter",
        "value": "1"
      }
    },
    {
      "set": {
        "tag": "copy counter to new",
        "field": "new_counter",
        "copy_from": "counter"
      }
    },
    {
      "remove": {
        "tag": "remove old counter",
        "field": "counter"
      }
    },
    {
      "set": {
        "tag": "transfer counter back",
        "field": "counter",
        "copy_from": "new_counter"
      }
    },
    {
      "remove": {
        "tag": "remove counter again",
        "field": "counter"
      }
    },
    {
      "set": {
        "tag": "copy to new counter again",
        "field": "new_counter",
        "copy_from": "counter"
      }
    }
  ]
}
```

We ingest some data and find that it was sent to the failure store.
```console
POST my-datastream-ingest/_doc?pipeline=complicated-processor
{
    "@timestamp": "2025-04-21T00:00:00Z",
    "counter_name": "test"
}
```

```console-result
{
  "_index": ".fs-my-datastream-ingest-2025.05.09-000001",
  "_id": "HnTJs5YBwrYNjPmaFcri",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 1,
  "_primary_term": 1,
  "failure_store": "used"
}
```

On checking the failure, we can quickly identify the tagged processor that caused the problem.
```console
GET my-datastream-ingest::failures/_search
```

```console-result
{
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
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
        "_index": ".fs-my-datastream-ingest-2025.05.09-000001",
        "_id": "HnTJs5YBwrYNjPmaFcri",
        "_score": 1,
        "_source": {
          "@timestamp": "2025-05-09T06:41:24.775Z",
          "document": {
            "index": "my-datastream-ingest",
            "source": {
              "@timestamp": "2025-04-21T00:00:00Z",
              "counter_name": "test"
            }
          },
          "error": {
            "type": "illegal_argument_exception",
            "message": "field [counter] not present as part of path [counter]",
            "stack_trace": """j.l.IllegalArgumentException: field [counter] not present as part of path [counter]
	at o.e.i.IngestDocument.getFieldValue(IngestDocument.java:202)
	at o.e.i.c.SetProcessor.execute(SetProcessor.java:86)
	... 14 more
""",
            "pipeline_trace": [
              "complicated-processor"
            ],
            "pipeline": "complicated-processor",
            "processor_type": "set", <1>
            "processor_tag": "copy to new counter again" <2>
          }
        }
      }
    ]
  }
}
```
1. Helpful, but which set processor on the pipeline could it be?
2. The tag of the exact processor that the document failed on.

Without tags in place it would not be as clear where in the pipeline the indexing problem occurred. Tags provide a unique identifier for a processor that can be quickly referenced in case of an ingest failure.

## Alerting on failed ingestion [failure-store-examples-alerting]

Since failure stores can be searched just like a normal data stream, we can use them as inputs to [alerting rules](../../../explore-analyze/alerts-cases/alerts.md) in
{{kib}}. Here is a simple alerting example that is triggered when more than ten indexing failures have occurred in the last five minutes for a data stream:

:::::{stepper}

::::{step} Create a failure store data view
If you want to use KQL or Lucene query types, you should first create a data view for your failure store data.
If you plan to use {{esql}} or the Query DSL query types, this step is not required.

Navigate to the data view page in Kibana and add a new data view. Set the index pattern to your failure store using the selector syntax.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_create_data_view.png
:alt: create a data view using the failure store syntax in the index name
:::
::::

::::{step} Create new rule
Navigate to Management / Alerts and Insights / Rules. Create a new rule. Choose the {{es}} query option.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_create_rule.png
:alt: create a new alerting rule and select the elasticsearch query option
:::
::::

::::{step} Pick your query type
Choose which query type you wish to use

For KQL/Lucene queries, reference the data view that contains your failure store.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_kql.png
:alt: use the data view created in the previous step as the input to the kql query
:::

For Query DSL queries, use the `::failures` suffix on your data stream name.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_dsl.png
:alt: use the ::failures suffix in the data stream name in the query dsl
:::

For {{esql}} queries, use the `::failures` suffix on your data stream name in the `FROM` command.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_esql.png
:alt: use the ::failures suffix in the data stream name in the from command
:::
::::

::::{step} Test
Configure schedule, actions, and details of the alert before saving the rule.

:::{image} /manage-data/images/elasticsearch-reference-management_failure_store_alerting_finish.png
:alt: complete the rule configuration and save it
:::
::::

::::{step} Done
::::

:::::

## Data remediation [failure-store-examples-remediation]

If you've encountered a long span of ingestion failures you may find that a sizeable gap of events has appeared in your data stream. If the failure store is enabled, the documents that should fill those gaps would be tucked away in the data stream's failure store. Because failure stores are made up of regular indices and the failure documents contain the document source that failed, the failure documents can often times be replayed into your production data streams.

::::{warning}
Care should be taken when replaying data into a data stream from a failure store. Any failures during the replay process may generate new failures in the failure store which can duplicate and obscure the original events.
::::

We recommend a few best practices for remediating failure data.

**Separate your failures beforehand.** As described in the previous [failure document source](./failure-store.md#use-failure-store-document-source) section, failure documents are structured differently depending on when the document failed during ingestion. We recommend to separate documents by ingest pipeline failures and indexing failures at minimum. Ingest pipeline failures often need to have the original pipeline re-run, while index failures should skip any pipelines. Further separating failures by index or specific failure type may also be beneficial.

**Perform a failure store rollover.** Consider [rolling over the failure store](./failure-store.md#manage-failure-store-rollover) before attempting to remediate failures. This will create a new failure index that will collect any new failures during the remediation process.

**Use an ingest pipeline to convert failure documents back into their original document.** Failure documents store failure information along with the document that failed ingestion. The first step for remediating documents should be to use an ingest pipeline to extract the original source from the failure document and then discard any other information about the failure.

**Simulate first to avoid repeat failures.** If you must run a pipeline as part of your remediation process, it is best to simulate the pipeline against the failure first. This will catch any unforeseen issues that may fail the document a second time. Remember, ingest pipeline failures will capture the document before an ingest pipeline is applied to it, which can further complicate remediation when a failure document becomes nested inside a new failure. The easiest way to simulate these changes is via the [pipeline simulate API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ingest-simulate) or the [simulate ingest API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-simulate-ingest).

### Remediating ingest node failures [failure-store-examples-remediation-ingest]

Failures that occurred during ingest processing will be stored as they were before any pipelines were run. To replay the document into the data stream we will need to re-run any applicable pipelines for the document.

:::::{stepper}

::::{step} Separate out which failures to replay

Start off by constructing a query that can be used to consistently identify which failures will be remediated.

```console
POST my-datastream-ingest-example::failures/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "exists": { <1>
            "field": "error.pipeline"
          }
        },
        {
          "match": { <2>
            "document.index": "my-datastream-ingest-example"
          }
        },
        {
          "match": { <3>
            "error.type": "illegal_argument_exception"
          }
        },
        {
          "range": { <4>
            "@timestamp": {
              "gt": "2025-05-01T00:00:00Z",
              "lte": "2025-05-02T00:00:00Z"
            }
          }
        }
      ]
    }
  }
}
```
1. Require the `error.pipeline` field to exist. This filters to ingest pipeline failures only.
2. Filter on the data stream name to remediate documents headed for a specific index.
3. Further narrow which kind of failure you are attempting to remediate. In this example we are targeting a specific type of error.
4. Filter on timestamp to only retrieve failures before a certain point in time. This provides a stable set of documents.

Take note of the documents that are returned. We can use these to simulate that our remediation logic makes sense
```console-result
{
  "took": 14,
  "timed_out": false,
  "_shards": {
    "total": 2,
    "successful": 2,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 2.575364,
    "hits": [
      {
        "_index": ".fs-my-datastream-ingest-example-2025.05.16-000001",
        "_id": "cOnR2ZYByIwDXH-g6GpR",
        "_score": 2.575364,
        "_source": {
          "@timestamp": "2025-05-01T15:58:53.522Z", <1>
          "document": {
            "index": "my-datastream-ingest-example",
            "source": {
              "@timestamp": "2025-05-01T00:00:00Z",
              "data": {
                "counter": 42 <2>
              }
            }
          },
          "error": {
            "type": "illegal_argument_exception",
            "message": "field [id] not present as part of path [data.id]", <3>
            "stack_trace": """j.l.IllegalArgumentException: field [id] not present as part of path [data.id]
	at o.e.i.IngestDocument.getFieldValue(IngestDocument.java:202)
	at o.e.i.c.SetProcessor.execute(SetProcessor.java:86)
	... 14 more
""",
            "pipeline_trace": [
              "my-datastream-default-pipeline"
            ],
            "pipeline": "my-datastream-default-pipeline", <4>
            "processor_type": "set"
          }
        }
      }
    ]
  }
}
```
1. This document is what we'll use for our simulations.
2. It had a counter value.
3. The document was missing a required field.
4. The document failed in the `my-data-stream-default-pipeline`
   ::::

::::{step} Fix the original problem
Because ingest pipeline failures need to be reprocessed by their original pipelines, any problems with those pipelines should be fixed before remediating failures. Investigating the pipeline mentioned in the example above shows that there is a processor that expects a field to be present that is not always present.

```console-result
{
  "my-datastream-default-pipeline": {
    "processors": [
      {
        "set": { <1>
          "field": "identifier",
          "copy_from": "data.id"
        }
      }
    ]
  }
}
```
1. The `data.id` field is expected to be present. If it isn't present this pipeline will fail.

Fixing a failure's root cause is a often a bespoke process. In this example, instead of discarding the data, we will make this identifier field optional.

```console
PUT _ingest/pipeline/my-datastream-default-pipeline
{
  "processors": [
    {
      "set": {
        "field": "identifier",
        "copy_from": "data.id",
        "if": "ctx.data?.id != null" <1>
      }
    }
  ]
}
```
1. Conditionally run the processor only if the field exists.

::::

::::{step} Create a pipeline to convert failure documents

We must convert our failure documents back into their original forms and send them off to be reprocessed. We will create a pipeline to do this:

```console
PUT _ingest/pipeline/my-datastream-remediation-pipeline
{
  "processors": [
    {
      "script": {
      "lang": "painless",
      "source": """
          ctx._index = ctx.document.index; <1>
          ctx._routing = ctx.document.routing;
          def s = ctx.document.source; <2>
          ctx.remove("error"); <3>
          ctx.remove("document"); <4>
          for (e in s.entrySet()) { <5>
            ctx[e.key] = e.value;
          }"""
      }
    },
    {
      "reroute": { <6>
        "destination": "my-datastream-ingest-example"
      }
    }
  ]
}
```
1. Copy the original index name from the failure document over into the document's metadata. If you use custom document routing, copy that over too.
2. Capture the source of the original document.
3. Discard the `error` field since it won't be needed for the remediation.
4. Also discard the `document` field.
5. We extract all the fields from the original document's source back to the root of the document.
6. Since the pipeline that failed was the default pipeline on `my-datastream-ingest-example`, we will use the `reroute` processor to send any remediated documents to that data stream's default pipeline again to be reprocessed.

::::

::::{step} Test your pipelines
Before sending data off to be reindexed, be sure to test the pipelines in question with an example document to make sure they work. First, test to make sure the resulting document from the remediation pipeline is shaped how you expect. We can use the [simulate pipeline API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ingest-simulate) for this.

```console
POST _ingest/pipeline/_simulate
{
  "pipeline": { <1>
    "processors": [
      {
        "script": {
        "lang": "painless",
        "source": """
            ctx._index = ctx.document.index;
            ctx._routing = ctx.document.routing;
            def s = ctx.document.source;
            ctx.remove("error");
            ctx.remove("document");
            for (e in s.entrySet()) {
              ctx[e.key] = e.value;
            }"""
        }
      },
      {
        "reroute": {
          "destination": "my-datastream-ingest-example"
        }
      }
    ]
  },
  "docs": [ <2>
    {
        "_index": ".fs-my-datastream-ingest-example-2025.05.16-000001",
        "_id": "cOnR2ZYByIwDXH-g6GpR",
        "_source": {
          "@timestamp": "2025-05-01T15:58:53.522Z",
          "document": {
            "index": "my-datastream-ingest-example",
            "source": {
              "@timestamp": "2025-05-01T00:00:00Z",
              "data": {
                "counter": 42
              }
            }
          },
          "error": {
            "type": "illegal_argument_exception",
            "message": "field [id] not present as part of path [data.id]",
            "stack_trace": """j.l.IllegalArgumentException: field [id] not present as part of path [data.id]
	at o.e.i.IngestDocument.getFieldValue(IngestDocument.java:202)
	at o.e.i.c.SetProcessor.execute(SetProcessor.java:86)
	... 14 more
""",
            "pipeline_trace": [
              "my-datastream-default-pipeline"
            ],
            "pipeline": "my-datastream-default-pipeline",
            "processor_type": "set"
          }
        }
      }
  ]
}
```
1. The contents of the remediation pipeline written in the previous step.
2. The contents of an example failure document we identified in the previous steps.

```console-result
{
  "docs": [
    {
      "doc": {
        "_index": "my-datastream-ingest-example", <1>
        "_version": "-3",
        "_id": "cOnR2ZYByIwDXH-g6GpR", <2>
        "_source": { <3>
          "data": {
            "counter": 42
          },
          "@timestamp": "2025-05-01T00:00:00Z"
        },
        "_ingest": {
          "timestamp": "2025-05-01T20:58:03.566210529Z"
        }
      }
    }
  ]
}
```
1. The index has been updated via the reroute processor.
2. The document ID has stayed the same.
3. The source should cleanly match the contents of the original document.

Now that the remediation pipeline has been tested, be sure to test the end-to-end ingestion to verify that no further problems will arise. To do this, we will use the [simulate ingestion API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-simulate-ingest) to test multiple pipeline executions.

```console
POST _ingest/_simulate?pipeline=my-datastream-remediation-pipeline <1>
{
  "pipeline_substitutions": {
    "my-datastream-remediation-pipeline": { <2>
      "processors": [
        {
          "script": {
            "lang": "painless",
            "source": """
                ctx._index = ctx.document.index;
                ctx._routing = ctx.document.routing;
                def s = ctx.document.source;
                ctx.remove("error");
                ctx.remove("document");
                for (e in s.entrySet()) {
                  ctx[e.key] = e.value;
                }"""
          }
        },
        {
          "reroute": {
            "destination": "my-datastream-ingest-example"
          }
        }
      ]
    }
  },
  "docs": [ <3>
    {
        "_index": ".fs-my-datastream-ingest-example-2025.05.16-000001",
        "_id": "cOnR2ZYByIwDXH-g6GpR",
        "_source": {
          "@timestamp": "2025-05-01T15:58:53.522Z",
          "document": {
            "index": "my-datastream-ingest-example",
            "source": {
              "@timestamp": "2025-05-01T00:00:00Z",
              "data": {
                "counter": 42
              }
            }
          },
          "error": {
            "type": "illegal_argument_exception",
            "message": "field [id] not present as part of path [data.id]",
            "stack_trace": """j.l.IllegalArgumentException: field [id] not present as part of path [data.id]
	at o.e.i.IngestDocument.getFieldValue(IngestDocument.java:202)
	at o.e.i.c.SetProcessor.execute(SetProcessor.java:86)
	... 14 more
""",
            "pipeline_trace": [
              "my-datastream-default-pipeline"
            ],
            "pipeline": "my-datastream-default-pipeline",
            "processor_type": "set"
          }
        }
      }
  ]
}
```
1. Set the pipeline to be the remediation pipeline name, otherwise the default pipeline for the document's index is used.
2. The contents of the remediation pipeline in previous steps.
3. The contents of the previously identified example failure document.

```console-result
{
  "docs": [
    {
      "doc": {
        "_id": "cOnR2ZYByIwDXH-g6GpR",
        "_index": "my-datastream-ingest-example", <1>
        "_version": -3,
        "_source": { <2>
          "@timestamp": "2025-05-01T00:00:00Z",
          "data": {
            "counter": 42
          }
        },
        "executed_pipelines": [ <3>
          "my-datastream-remediation-pipeline",
          "my-datastream-default-pipeline"
        ]
      }
    }
  ]
}
```
1. The index name has been updated.
2. The source is as expected after the default pipeline has run.
3. Ensure that both the new remediation pipeline and the original default pipeline have successfully run.

::::

::::{step} Reindex the failure documents
Combine the remediation pipeline with the failure store query together in a [reindex operation](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-reindex) to replay the failures.

```console
POST _reindex
{
  "source": {
    "index": "my-datastream-ingest-example::failures", <1>
    "query": {
      "bool": { <2>
        "must": [
          {
            "exists": {
              "field": "error.pipeline"
            }
          },
          {
            "match": {
              "document.index": "my-datastream-ingest-example"
            }
          },
          {
            "match": {
              "error.type": "illegal_argument_exception"
            }
          },
          {
            "range": {
              "@timestamp": {
                "gt": "2025-05-01T00:00:00Z",
                "lte": "2025-05-17T00:00:00Z"
              }
            }
          }
        ]
      }
    }
  },
  "dest": {
    "index": "my-datastream-ingest-example", <3>
    "op_type": "create",
    "pipeline": "my-datastream-remediation-pipeline" <4>
  }
}
```
1. Read from the failure store.
2. Only reindex failure documents that match the ones we are replaying.
3. Set the destination to the data stream the failures originally were sent to.
4. Replace the pipeline with the remediation pipeline.

```console-result
{
  "took": 469,
  "timed_out": false,
  "total": 1,
  "updated": 0,
  "created": 1, <1>
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1,
  "throttled_until_millis": 0,
  "failures": []
}
```
1. The failures have been remediated.

:::{tip}
Since the failure store is enabled on this data stream, it would be wise to check it for any further failures from the reindexing process. Failures that happen at this point in the process may end up as nested failures in the failure store. Remediating nested failures can quickly become a hassle as the original document gets nested multiple levels deep in the failure document. For this reason, it is suggested to remediate data during a quiet period when no other failures are likely to arise. Furthermore, rolling over the failure store before executing the remediation would allow easier discarding of any new nested failures and only operate on the original failure documents.
:::

::::{step} Done
Once any failures have been remediated, you may wish to purge the failures from the failure store to clear up space and to avoid warnings about failed data that has already been replayed. Otherwise, your failures will stay available until the maximum failure store retention should you need to reference them.
::::

:::::

### Remediating mapping and shard failures [failure-store-examples-remediation-mapping]

As described in the previous [failure document source](./failure-store.md#use-failure-store-document-source) section, failures that occur due to a mapping or indexing issue will be stored as they were after any pipelines had executed. This means that to replay the document into the data stream we will need to make sure to skip any pipelines that have already run.

:::{tip}
You can greatly simplify this remediation process by writing any ingest pipelines to be idempotent. In that case, any document that has already be processed that passes through a pipeline again would be unchanged.
:::

:::::{stepper}

::::{step} Separate out which failures to replay

Start off by constructing a query that can be used to consistently identify which failures will be remediated.

```console
POST my-datastream-indexing-example::failures/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "exists": { <1>
            "field": "error.pipeline"
          }
        }
      ],
      "must": [
        {
          "match": { <2>
            "document.index": "my-datastream-indexing-example"
          }
        },
        {
          "match": { <3>
            "error.type": "document_parsing_exception"
          }
        },
        {
          "range": { <4>
            "@timestamp": {
              "gt": "2025-05-01T00:00:00Z",
              "lte": "2025-05-02T00:00:00Z"
            }
          }
        }
      ]
    }
  }
}
```
1. Require the `error.pipeline` field to not exist. This filters out any ingest pipeline failures, and only returns indexing failures.
2. Filter on the data stream name to remediate documents headed for a specific index.
3. Further narrow which kind of failure you are attempting to remediate. In this example we are targeting a specific type of error.
4. Filter on timestamp to only retrieve failures before a certain point in time. This provides a stable set of documents.

Take note of the documents that are returned. We can use these to simulate that our remediation logic makes sense.
```console-result
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 1.5753641,
    "hits": [
      {
        "_index": ".fs-my-datastream-indexing-example-2025.05.16-000002",
        "_id": "_lA-GJcBHLe506UUGL0I",
        "_score": 1.5753641,
        "_source": { <1>
          "@timestamp": "2025-05-02T18:53:31.153Z",
          "document": {
            "id": "_VA-GJcBHLe506UUFL2i",
            "index": "my-datastream-indexing-example",
            "source": {
              "processed": true, <2>
              "data": {
                "counter": 37
              }
            }
          },
          "error": {
            "type": "document_parsing_exception", <3>
            "message": "[1:40] failed to parse: data stream timestamp field [@timestamp] is missing",
            "stack_trace": """o.e.i.m.DocumentParsingException: [1:40] failed to parse: data stream timestamp field [@timestamp] is missing
	at o.e.i.m.DocumentParser.wrapInDocumentParsingException(DocumentParser.java:265)
	at o.e.i.m.DocumentParser.internalParseDocument(DocumentParser.java:162)
	... 19 more
Caused by: j.l.IllegalArgumentException: data stream timestamp field [@timestamp] is missing
	at o.e.i.m.DataStreamTimestampFieldMapper.extractTimestampValue(DataStreamTimestampFieldMapper.java:210)
	at o.e.i.m.DataStreamTimestampFieldMapper.postParse(DataStreamTimestampFieldMapper.java:223)
	... 20 more
"""
          }
        }
      }
    ]
  }
}
```
1. This document is what we'll use for our simulations.
2. The document was missing a required `@timestamp` field.
3. The document failed with a `document_parsing_exception` because of the missing timestamp.

::::

::::{step} Fix the original problem

There are a broad set of possible indexing failures. Most of these problems stem from incorrect values for a particular mapping. Sometimes a large number of new fields are dynamically mapped and the maximum number of mapping fields is reached, so no more can be added. In our example above, the document being indexed is missing a required timestamp.

These problems can occur in a number of places: Data sent from a client may be incomplete, ingest pipelines may not be producing the correct result, or the index mapping may need to be updated to account for changes in data.

Once all clients and pipelines are producing complete and correct documents, and your mappings are correctly configured for your incoming data, proceed with the remediation.

::::

::::{step} Create a pipeline to convert failure documents

We must convert our failure documents back into their original forms and send them off to be reprocessed. We will create a pipeline to do this. Since the example failure was due to not having a timestamp on the document, we will simply use the timestamp at the time of failure for the document since the original timestamp is missing. This solution assumes that the documents we are remediating were created very closely to when the failure occurred. Your remediation process may need adjustments if this is not applicable for you.

```console
PUT _ingest/pipeline/my-datastream-remediation-pipeline
{
  "processors": [
    {
      "script": {
      "lang": "painless",
      "source": """
          ctx._index = ctx.document.index; <1>
          ctx._routing = ctx.document.routing;
          def s = ctx.document.source; <2>
          ctx.remove("error"); <3>
          ctx.remove("document"); <4>
          for (e in s.entrySet()) { <5>
            ctx[e.key] = e.value;
          }"""
      }
    }
  ]
}
```
1. Copy the original index name from the failure document over into the document's metadata. If you use custom document routing, copy that over too.
2. Capture the source of the original document.
3. Discard the `error` field since it wont be needed for the remediation.
4. Also discard the `document` field.
5. We extract all the fields from the original document's source back to the root of the document. The `@timestamp` field is not overwritten and thus will be present in the final document.

:::{important}
Remember that a document that has failed during indexing has already been processed by the ingest processor! It shouldn't need to be processed again unless you made changes to your pipeline to fix the original problem. Make sure that any fixes applied to the ingest pipeline are reflected in the pipeline logic here.
:::

::::

::::{step} Test your pipeline
Before sending data off to be reindexed, be sure to test the remedial pipeline with an example document to make sure it works. Most importantly, make sure the resulting document from the remediation pipeline is shaped how you expect. We can use the [simulate pipeline API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ingest-simulate) for this.

```console
POST _ingest/pipeline/_simulate
{
  "pipeline": { <1>
    "processors": [
      {
        "script": {
        "lang": "painless",
        "source": """
            ctx._index = ctx.document.index;
            ctx._routing = ctx.document.routing;
            def s = ctx.document.source;
            ctx.remove("error");
            ctx.remove("document");
            for (e in s.entrySet()) {
              ctx[e.key] = e.value;
            }"""
        }
      }
    ]
  },
  "docs": [ <2>
    {
        "_index": ".fs-my-datastream-indexing-example-2025.05.16-000002",
        "_id": "_lA-GJcBHLe506UUGL0I",
        "_score": 1.5753641,
        "_source": {
          "@timestamp": "2025-05-02T18:53:31.153Z",
          "document": {
            "id": "_VA-GJcBHLe506UUFL2i",
            "index": "my-datastream-indexing-example",
            "source": {
              "processed": true,
              "data": {
                "counter": 37
              }
            }
          },
          "error": {
            "type": "document_parsing_exception",
            "message": "[1:40] failed to parse: data stream timestamp field [@timestamp] is missing",
            "stack_trace": """o.e.i.m.DocumentParsingException: [1:40] failed to parse: data stream timestamp field [@timestamp] is missing
	at o.e.i.m.DocumentParser.wrapInDocumentParsingException(DocumentParser.java:265)
	at o.e.i.m.DocumentParser.internalParseDocument(DocumentParser.java:162)
	... 19 more
Caused by: j.l.IllegalArgumentException: data stream timestamp field [@timestamp] is missing
	at o.e.i.m.DataStreamTimestampFieldMapper.extractTimestampValue(DataStreamTimestampFieldMapper.java:210)
	at o.e.i.m.DataStreamTimestampFieldMapper.postParse(DataStreamTimestampFieldMapper.java:223)
	... 20 more
"""
          }
        }
      }
  ]
}
```
1. The contents of the remediation pipeline written in the previous step.
2. The contents of an example failure document we identified in the previous steps.

```console-result
{
  "docs": [
    {
      "doc": {
        "_index": "my-datastream-indexing-example", <1>
        "_version": "-3",
        "_id": "_lA-GJcBHLe506UUGL0I",
        "_source": { <2>
          "processed": true,
          "@timestamp": "2025-05-28T18:53:31.153Z", <3>
          "data": {
            "counter": 37
          }
        },
        "_ingest": {
          "timestamp": "2025-05-28T19:14:50.457560845Z"
        }
      }
    }
  ]
}
```
1. The index has been updated via the script processor.
2. The source should reflect any fixes and match the expected document shape for the final index.
3. In this example case, we find that the failure timestamp has stayed in the source.

::::

::::{step} Reindex the failure documents
Combine the remediation pipeline with the failure store query together in a [reindex operation](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-reindex) to replay the failures.

```console
POST _reindex
{
  "source": {
    "index": "my-datastream-indexing-example::failures", <1>
    "query": {
      "bool": { <2>
        "must_not": [
          {
            "exists": {
              "field": "error.pipeline"
            }
          }
        ],
        "must": [
          {
            "match": {
              "document.index": "my-datastream-indexing-example"
            }
          },
          {
            "match": {
              "error.type": "document_parsing_exception"
            }
          },
          {
            "range": {
              "@timestamp": {
                "gt": "2025-05-01T00:00:00Z",
                "lte": "2025-05-28T19:00:00Z"
              }
            }
          }
        ]
      }
    }
  },
  "dest": {
    "index": "my-datastream-indexing-example", <3>
    "op_type": "create",
    "pipeline": "my-datastream-remediation-pipeline" <4>
  }
}
```
1. Read from the failure store.
2. Only reindex failure documents that match the ones we are replaying.
3. Set the destination to the data stream the failures originally were sent to. The remediation pipeline in the example updates the index to be the correct one, but a destination is still required.
4. Replace the original pipeline with the remediation pipeline. This will keep any default pipelines from running.

```console-result
{
  "took": 469,
  "timed_out": false,
  "total": 1,
  "updated": 0,
  "created": 1, <1>
  "deleted": 0,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1,
  "throttled_until_millis": 0,
  "failures": []
}
```
1. The failures have been remediated.

:::{tip}
Since the failure store is enabled on this data stream, it would be wise to check it for any further failures from the reindexing process. Failures that happen at this point in the process may end up as nested failures in the failure store. Remediating nested failures can quickly become a hassle as the original document gets nested multiple levels deep in the failure document. For this reason, it is suggested to remediate data during a quiet period where no other failures will arise. Furthermore, rolling over the failure store before executing the remediation would allow easier discarding of any new nested failures and only operate on the original failure documents.
:::

::::{step} Done
Once any failures have been remediated, you may wish to purge the failures from the failure store to clear up space and to avoid warnings about failed data that has already been replayed. Otherwise, your failures will stay available until the maximum failure store retention should you need to reference them.
::::

:::::
