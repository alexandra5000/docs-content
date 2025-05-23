---
mapped_pages:
  - https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-classify-text.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: machine-learning
---

# Classify text [ml-nlp-classify-text]

These NLP tasks enable you to identify the language of text and classify or label unstructured input text:

* [{{lang-ident-cap}}](ml-nlp-lang-ident.md)
* [Text classification](#ml-nlp-text-classification)
* [Zero-shot text classification](#ml-nlp-zero-shot)

## {{lang-ident-cap}} [_lang_ident_cap]

The {{lang-ident}} model is provided out-of-the box in your {{es}} cluster. You can find the documentation of the model on the [{{lang-ident-cap}}](ml-nlp-lang-ident.md) page under the Built-in models section.

## Text classification [ml-nlp-text-classification]

Text classification assigns the input text to one of multiple classes that best describe the text. The classes used depend on the model and the data set that was used to train it. Based on the number of classes, two main types of classification exist: binary classification, where the number of classes is exactly two, and multi-class classification, where the number of classes is more than two.

This task can help you analyze text for markers of positive or negative sentiment or classify text into various topics. For example, you might use a trained model to perform sentiment analysis and determine whether the following text is "POSITIVE" or "NEGATIVE":

```js
{
    docs: [{"text_field": "This was the best movie I’ve seen in the last decade!"}]
}
...
```

Likewise, you might use a trained model to perform multi-class classification and determine whether the following text is a news topic related to "SPORTS", "BUSINESS", "LOCAL", or "ENTERTAINMENT":

```js
{
    docs: [{"text_field": "The Blue Jays played their final game in Toronto last night and came out with a win over the Yankees, highlighting just how far the team has come this season."}]
}
...
```

## Zero-shot text classification [ml-nlp-zero-shot]

The zero-shot classification task offers the ability to classify text without training a model on a specific set of classes. Instead, you provide the classes when you deploy the model or at {{infer}} time. It uses a model trained on a large data set that has gained a general language understanding and asks the model how well the labels you provided fit with your text.

This task enables you to analyze and classify your input text even when you don’t have sufficient training data to train a text classification model.

For example, you might want to perform multi-class classification and determine whether a news topic is related to "SPORTS", "BUSINESS", "LOCAL", or "ENTERTAINMENT". However, in this case the model is not trained specifically for news classification; instead, the possible labels are provided together with the input text at {{infer}} time:

```js
{
    docs: [{"text_field": "The S&P 500 gained a meager 12 points in the day’s trading. Trade volumes remain consistent with those of the past week while investors await word from the Fed about possible rate increases."}],
    "inference_config": {
        "zero_shot_classification": {
            "labels": ["SPORTS", "BUSINESS", "LOCAL", "ENTERTAINMENT"]
        }
    }
}
```

The task returns the following result:

```js
...
{
    "predicted_value": "BUSINESS"
    ...
}
...
```

You can use the same model to perform {{infer}} with different classes, such as:

```js
{
    docs: [{"text_field": "Hello support team. I’m writing to inquire about the possibility of sending my broadband router in for repairs. The internet is really slow and the router keeps rebooting! It’s a big problem because I’m in the middle of binge-watching The Mandalorian!"}]
    "inference_config": {
        "zero_shot_classification": {
            "labels": ["urgent", "internet", "phone", "cable", "mobile", "tv"]
        }
    }
}
```

The task returns the following result:

```js
...
{
    "predicted_value": ["urgent", "internet", "tv"]
    ...
}
...
```

Since you can adjust the labels while you perform {{infer}}, this type of task is exceptionally flexible. If you are consistently using the same labels, however, it might be better to use a fine-tuned text classification model.
