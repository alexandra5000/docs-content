---
mapped_pages:
  - https://www.elastic.co/guide/en/machine-learning/current/hyperparameters.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: machine-learning
---

# Hyperparameter optimization [hyperparameters]

When you create a {{dfanalytics-job}} for {{classification}} or {{reganalysis}}, there are advanced configuration options known as *hyperparameters*. The ideal hyperparameter values vary from one data set to another. Therefore, by default the job calculates the best combination of values through a process of *hyperparameter optimization*.

Hyperparameter optimization involves multiple rounds of analysis. Each round involves a different combination of hyperparameter values, which are determined through a combination of random search and Bayesian optimization techniques. If you explicitly set a hyperparameter, that value is not optimized and remains the same in each round. To determine which round produces the best results, stratified K-fold cross-validation methods are used to split the data set, train a model, and calculate its performance on validation data.

You can view the hyperparameter values that were ultimately chosen by expanding the job details in {{kib}} or by using the [get trained models API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-get-trained-models). You can also see the specific type of validation loss (such as mean squared error or binomial cross entropy) that was used to compare each round of optimization using the [get {{dfanalytics-job}} stats API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-get-data-frame-analytics-stats).

Different hyperparameters may affect the model performance to a different degree. To estimate the importance of the optimized hyperparameters, analysis of variance decomposition is used. The resulting `absolute importance` shows how much the variation of a hyperparameter impacts the variation in the validation loss. Additionally, `relative importance` is also computed which gives the importance of the hyperparameter compared to the rest of the tuneable hyperparameters. The sum of all relative importances is 1. You can check these results in the response of the [get {{dfanalytics-job}} stats API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-ml-get-data-frame-analytics-stats).

::::{tip}
Unless you fully understand the purpose of a hyperparameter, it is highly recommended that you leave it unset and allow hyperparameter optimization to occur.
::::
