---
navigation_title: End-to-end tutorial
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/nlp-example.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: elasticsearch
---

# An NLP end-to-end tutorial [nlp-example]

This guide focuses on a concrete task: getting a machine learning trained model loaded into Elasticsearch and set up to enrich your documents.

Elasticsearch supports many different ways to use machine learning models. In this guide, we will use a trained model to enrich documents at ingest time using ingest pipelines configured within Kibana’s **Content** UI.

In this guide, we’ll accomplish the above using the following steps:

* **Set up a Cloud deployment**: We will use Elastic Cloud to host our deployment, as it makes it easy to scale machine learning nodes.
* **Load a model with Eland**: We will use the Eland Elasticsearch client to import our chosen model into Elasticsearch. Once we’ve verified that the model is loaded, we will be able to use it in an ingest pipeline.
* **Setup an ML inference pipeline**: We will create an Elasticsearch index with a predefined mapping and add an inference pipeline.
* **Show enriched results**: We will ingest some data into our index and observe that the pipeline enriches our documents.

Follow the instructions to load a text classification model and set it up to enrich some photo comment data. Once you’re comfortable with the steps involved, use this guide as a blueprint for working with other machine learning trained models.

**Table of contents**:

* [Create an {{ecloud}} deployment](#nlp-example-cloud-deployment)
* [Clone Eland](#nlp-example-clone-eland)
* [Deploy the trained model](#nlp-example-deploy-model)
* [Create an index and define an ML inference pipeline](#nlp-example-create-index-and-define-ml-inference-pipeline)
* [Index documents](#nlp-example-index-documents)
* [Summary](#nlp-example-summary)
* [Learn more](#nlp-example-learn-more)

## Create an {{ecloud}} deployment [nlp-example-cloud-deployment]

Your deployment will need a machine learning instance to upload and deploy trained models.

If your team already has an Elastic Cloud deployment, make sure it has at least one machine learning instance. If it does not, **Edit** your deployment to add capacity. For this tutorial, we’ll need at least 2GB of RAM on a single machine learning instance.

If your team does not have an Elastic Cloud deployment, start by signing up for a [free Elastic Cloud trial](https://cloud.elastic.co/registration). After creating an account, you’ll have an active subscription and you’ll be prompted to create your first deployment.

Follow the steps to **Create** a new deployment. Make sure to add capacity to the **Machine Learning instances** under the **Advanced settings** before creating the deployment. To simplify scaling, turn on the **Autoscale this deployment** feature. If you use autoscaling, you should increase the minimum RAM for the machine learning instance. For this tutorial, we’ll need at least 2GB of RAM. For more details, refer to [Create a deployment^](../../../deploy-manage/deploy/elastic-cloud/create-an-elastic-cloud-hosted-deployment.md) in the Elastic Cloud documentation.

## Clone Eland [nlp-example-clone-eland]

Elastic’s [Eland](https://github.com/elastic/eland) tool makes it easy to upload trained models to your deployment via Docker.

Eland is a specialized Elasticsearch client for exploring and manipulating data, which we can use to upload trained models into Elasticsearch.

To clone and build Eland using Docker, run the following commands:

```sh
git clone git@github.com:elastic/eland.git
cd eland
docker build -t elastic/eland .
```

## Deploy the trained model [nlp-example-deploy-model]

Now that you have a deployment and a way to upload models, you will need to choose a trained model that fits your data. [Hugging Face](https://huggingface.co/) has a large repository of publicly available trained models. The model you choose will depend on your data and what you would like to do with it.

For the purposes of this guide, let’s say we have a data set of photo comments. In order to promote a positive atmosphere on our platform, we’d like the first few comments on each photo to be positive comments. For this task, the [`distilbert-base-uncased-finetuned-sst-2-english`](https://huggingface.co/distilbert-base-uncased-finetuned-sst-2-english?text=I+like+you.+I+love+you) model is a good fit.

To upload this model to your deployment, you need a few pieces of data:

* The deployment URL. You can get this via the **Copy endpoint** link next to **Elasticsearch** on the deployment management screen. It will look like `https://ml-test.es.us-west1.gcp.cloud.es.io:443`. Make sure to append the port if it isn’t present, as Eland requires the URL to have a scheme, host, and port. 443 is the default port for HTTPS.
* The deployment username and password for your deployment. This is displayed one time when the deployment is created. It will look like `elastic` and `xUjaFNTyycG34tQx5Iq9JIIA`.
* The trained model id. This comes from Hugging Face. It will look like `distilbert-base-uncased-finetuned-sst-2-english`.
* The trained model task type. This is the kind of machine learning task the model is designed to achieve. It will be one of: `fill_mask`, `ner`, `text_classification`, `text_embedding`, and `zero_shot_classification`. For our use case, we will use `text_classification`.

We can now upload our chosen model to Elasticsearch by providing these options to Eland.

```sh
docker run -it --rm --network host \
    elastic/eland \
    eland_import_hub_model \
      --url https://ml-test.es.us-west1.gcp.cloud.es.io:443 \
      -u elastic -p <PASSWORD> \
      --hub-model-id distilbert-base-uncased-finetuned-sst-2-english \
      --task-type text_classification \
      --start
```

This script should take roughly 2-3 minutes to run. Once your model has been successfully deployed to your Elastic deployment, navigate to Kibana’s **Trained Models** page to verify it is ready. You can find this page under **Machine Learning > Analytics** menu and then **Trained Models > Model Management**. If you do not see your model in the list, you may need to click **Synchronize your jobs and trained models**. Your model is now ready to be used.

## Create an index and define an ML inference pipeline [nlp-example-create-index-and-define-ml-inference-pipeline]

We are now ready to use Kibana’s **Content** UI to enrich our documents with inference data. Before we ingest photo comments into Elasticsearch, we will first create an ML inference pipeline. The pipeline will enrich the incoming photo comments with inference data indicating if the comments are positive.

Let’s say our photo comments look like this when they are uploaded as a document into Elasticsearch:

```js
{
  "photo_id": "78sdv71-8vdkjaj-knew629-vc8459p",
  "body": "your dog is so cute!",
  ...
}
```

We want to run our documents through an inference processor that uses the trained model we uploaded to determine if the comments are positive. To do this, we first need to set up an Elasticsearch index.

* From the Kibana home page, start by clicking the Search card.
* Click the button to **Create an Elasticsearch index**.
* Choose to **Use the API** and give your index a name. It will automatically be prefixed with `search-`. For this demo, we will name the index `search-photo-comments`.
* After clicking **Create Index**, you will be redirected to the overview page for your new index.

To configure the ML inference pipeline, we need the index to have an existing field mapping so we can choose which field to analyze. This can be done via the [index mapping API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-put-mapping) in the Kibana Dev Tools or simply through a cURL command:

```js
PUT search-photo-comments/_mapping
{
  "properties": {
    "photo_id": { "type": "keyword" },
    "body": { "type": "text" }
  }
}
```

Now it’s time to create an inference pipeline.

1. From the overview page for your `search-photo-comments` index in "Search", click the **Pipelines** tab. By default, Elasticsearch does not create any index-specific ingest pipelines.
2. Because we want to customize these pipelines, we need to **Copy and customize** the `search-default-ingestion` ingest pipeline. Find this option above the settings for the `search-default-ingestion` ingest pipeline. This will create two new index-specific ingest pipelines.

Next, we’ll add an inference pipeline.

1. Locate the section **Machine Learning Inference Pipelines**, then select **Add inference pipeline**.
2. Give your inference pipeline a name, select the trained model we uploaded, and select the `body` field to be analyzed.
3. Optionally, choose a field name to store the output. We’ll call it `positivity_result`.

You can also run example documents through a simulator and review the pipeline before creating it.

## Index documents [nlp-example-index-documents]

At this point, everything is ready to enrich documents at index time.

From the Kibana Dev Console, or simply using a cURL command, we can index a document. We’ll use a `_run_ml_inference` flag to tell the `search-photo-comments` pipeline to run the index-specific ML inference pipeline that we created. This field will not be indexed in the document.

```js
POST search-photo-comments/_doc/my-new-doc?pipeline=search-photo-comments
{
  "photo_id": "78sdv71-8vdkjaj-knew629-vc8459p",
  "body": "your dog is so cute!",
  "_run_ml_inference": true
}
```

Once the document is indexed, use the API to retrieve it and view the enriched data.

```js
GET search-photo-comments/_doc/my-new-doc
```

```js
{
  "_index": "search-photo-comments",
  "_id": "_MQggoQBKYghsSwHbDvG",
  ...
  "_source": {
    ...
    "photo_id": "78sdv71-8vdkjaj-knew629-vc8459p",
    "body": "your dog is so cute!",
    "ml": {
      "inference": {
        "positivity_result": {
          "predicted_value": "POSITIVE",
          "prediction_probability": 0.9998022925461774,
          "model_id": "distilbert-base-uncased-finetuned-sst-2-english"
        }
      }
    }
  }
}
```

The document has new fields with the enriched data. The `ml.inference.positivity_result` field is an object with the analysis from the machine learning model. The model we used predicted with 99.98% confidence that the analyzed text is positive.

From here, we can write search queries to boost on `ml.inference.positivity_result.predicted_value`. This field will also be stored in a top-level `positivity_result` field if the model was confident enough.

## Summary [nlp-example-summary]

In this guide, we covered how to:

* Set up a deployment on Elastic Cloud with a machine learning instance.
* Deploy a machine learning trained model using the Eland Elasticsearch client.
* Configure an inference pipeline to use the trained model with Elasticsearch.
* Enrich documents with inference results from the trained model at ingest time.
* Query your search engine and sort by `positivity_result`.

## Learn more [nlp-example-learn-more]

* [Compatible third party models](ml-nlp-model-ref.md)
* [NLP Overview](ml-nlp-overview.md)
* [Docker section of Eland readme](https://github.com/elastic/eland#docker)
* [Deploying a model ML guide](ml-nlp-deploy-models.md)
* [Eland Authentication methods](ml-nlp-import-model.md#ml-nlp-authentication)
* [Adding inference pipelines](../machine-learning-in-kibana/inference-processing.md#ingest-pipeline-search-inference-add-inference-processors)
