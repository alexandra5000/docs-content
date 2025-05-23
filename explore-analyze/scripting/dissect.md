---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/dissect.html
applies_to:
  stack: ga
  serverless: ga
products:
  - id: elasticsearch
---

# Dissecting data [dissect]

Dissect matches a single text field against a defined pattern. A dissect pattern is defined by the parts of the string you want to discard. Paying special attention to each part of a string helps to build successful dissect patterns.

If you don’t need the power of regular expressions, use dissect patterns instead of grok. Dissect uses a much simpler syntax than grok and is typically faster overall. The syntax for dissect is transparent: tell dissect what you want and it will return those results to you.

## Dissect patterns [dissect-syntax]

Dissect patterns are comprised of *variables* and *separators*. Anything defined by a percent sign and curly braces `%{}` is considered a variable, such as `%{{clientip}}`. You can assign variables to any part of data in a field, and then return only the parts that you want. Separators are any values between variables, which could be spaces, dashes, or other delimiters.

For example, let’s say you have log data with a `message` field that looks like this:

```js
"message" : "247.37.0.0 - - [30/Apr/2020:14:31:22 -0500] \"GET /images/hm_nbg.jpg HTTP/1.0\" 304 0"
```

You assign variables to each part of the data to construct a successful dissect pattern. Remember, tell dissect *exactly* what you want you want to match on.

The first part of the data looks like an IP address, so you can assign a variable like `%{{clientip}}`. The next two characters are dashes with a space on either side. You can assign a variable for each dash, or a single variable to represent the dashes and spaces. Next are a set of brackets containing a timestamp. The brackets are a separator, so you include those in the dissect pattern. Thus far, the data and matching dissect pattern look like this:

```js
247.37.0.0 - - [30/Apr/2020:14:31:22 -0500]  <1>

%{clientip} %{ident} %{auth} [%{@timestamp}] <2>
```

1. The first chunks of data from the `message` field
2. Dissect pattern to match on the selected data chunks


Using that same logic, you can create variables for the remaining chunks of data. Double quotation marks are separators, so include those in your dissect pattern. The pattern replaces `GET` with a `%{{verb}}` variable, but keeps `HTTP` as part of the pattern.

```js
\"GET /images/hm_nbg.jpg HTTP/1.0\" 304 0

"%{verb} %{request} HTTP/%{httpversion}" %{response} %{size}
```

Combining the two patterns results in a dissect pattern that looks like this:

```js
%{clientip} %{ident} %{auth} [%{@timestamp}] \"%{verb} %{request} HTTP/%{httpversion}\" %{status} %{size}
```

Now that you have a dissect pattern, how do you test and use it?


## Test dissect patterns with Painless [dissect-patterns-test]

You can incorporate dissect patterns into Painless scripts to extract data. To test your script, use either the [field contexts](elasticsearch://reference/scripting-languages/painless/painless-api-examples.md#painless-execute-runtime-field-context) of the Painless execute API or create a runtime field that includes the script. Runtime fields offer greater flexibility and accept multiple documents, but the Painless execute API is a great option if you don’t have write access on a cluster where you’re testing a script.

For example, test your dissect pattern with the Painless execute API by including your Painless script and a single document that matches your data. Start by indexing the `message` field as a `wildcard` data type:

```console
PUT my-index
{
  "mappings": {
    "properties": {
      "message": {
        "type": "wildcard"
      }
    }
  }
}
```

If you want to retrieve the HTTP response code, add your dissect pattern to a Painless script that extracts the `response` value. To extract values from a field, use this function:

```painless
`.extract(doc["<field_name>"].value)?.<field_value>`
```

In this example, `message` is the `<field_name>` and `response` is the `<field_value>`:

```console
POST /_scripts/painless/_execute
{
  "script": {
    "source": """
      String response=dissect('%{clientip} %{ident} %{auth} [%{@timestamp}] "%{verb} %{request} HTTP/%{httpversion}" %{response} %{size}').extract(doc["message"].value)?.response;
        if (response != null) emit(Integer.parseInt(response)); <1>
    """
  },
  "context": "long_field", <2>
  "context_setup": {
    "index": "my-index",
    "document": {          <3>
      "message": """247.37.0.0 - - [30/Apr/2020:14:31:22 -0500] "GET /images/hm_nbg.jpg HTTP/1.0" 304 0"""
    }
  }
}
```

1. Runtime fields require the `emit` method to return values.
2. Because the response code is an integer, use the `long_field` context.
3. Include a sample document that matches your data.


The result includes the HTTP response code:

```console-result
{
  "result" : [
    304
  ]
}
```


## Use dissect patterns and scripts in runtime fields [dissect-patterns-runtime]

If you have a functional dissect pattern, you can add it to a runtime field to manipulate data. Because runtime fields don’t require you to index fields, you have incredible flexibility to modify your script and how it functions. If you already [tested your dissect pattern](#dissect-patterns-test) using the Painless execute API, you can use that *exact* Painless script in your runtime field.

To start, add the `message` field as a `wildcard` type like in the previous section, but also add `@timestamp` as a `date` in case you want to operate on that field for [other use cases](common-script-uses.md):

```console
PUT /my-index/
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "format": "strict_date_optional_time||epoch_second",
        "type": "date"
      },
      "message": {
        "type": "wildcard"
      }
    }
  }
}
```

If you want to extract the HTTP response code using your dissect pattern, you can create a runtime field like `http.response`:

```console
PUT my-index/_mappings
{
  "runtime": {
    "http.response": {
      "type": "long",
      "script": """
        String response=dissect('%{clientip} %{ident} %{auth} [%{@timestamp}] "%{verb} %{request} HTTP/%{httpversion}" %{response} %{size}').extract(doc["message"].value)?.response;
        if (response != null) emit(Integer.parseInt(response));
      """
    }
  }
}
```

After mapping the fields you want to retrieve, index a few records from your log data into {{es}}. The following request uses the [bulk API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-bulk) to index raw log data into `my-index`:

```console
POST /my-index/_bulk?refresh=true
{"index":{}}
{"timestamp":"2020-04-30T14:30:17-05:00","message":"40.135.0.0 - - [30/Apr/2020:14:30:17 -0500] \"GET /images/hm_bg.jpg HTTP/1.0\" 200 24736"}
{"index":{}}
{"timestamp":"2020-04-30T14:30:53-05:00","message":"232.0.0.0 - - [30/Apr/2020:14:30:53 -0500] \"GET /images/hm_bg.jpg HTTP/1.0\" 200 24736"}
{"index":{}}
{"timestamp":"2020-04-30T14:31:12-05:00","message":"26.1.0.0 - - [30/Apr/2020:14:31:12 -0500] \"GET /images/hm_bg.jpg HTTP/1.0\" 200 24736"}
{"index":{}}
{"timestamp":"2020-04-30T14:31:19-05:00","message":"247.37.0.0 - - [30/Apr/2020:14:31:19 -0500] \"GET /french/splash_inet.html HTTP/1.0\" 200 3781"}
{"index":{}}
{"timestamp":"2020-04-30T14:31:22-05:00","message":"247.37.0.0 - - [30/Apr/2020:14:31:22 -0500] \"GET /images/hm_nbg.jpg HTTP/1.0\" 304 0"}
{"index":{}}
{"timestamp":"2020-04-30T14:31:27-05:00","message":"252.0.0.0 - - [30/Apr/2020:14:31:27 -0500] \"GET /images/hm_bg.jpg HTTP/1.0\" 200 24736"}
{"index":{}}
{"timestamp":"2020-04-30T14:31:28-05:00","message":"not a valid apache log"}
```

You can define a simple query to run a search for a specific HTTP response and return all related fields. Use the `fields` parameter of the search API to retrieve the `http.response` runtime field.

```console
GET my-index/_search
{
  "query": {
    "match": {
      "http.response": "304"
    }
  },
  "fields" : ["http.response"]
}
```

Alternatively, you can define the same runtime field but in the context of a search request. The runtime definition and the script are exactly the same as the one defined previously in the index mapping. Just copy that definition into the search request under the `runtime_mappings` section and include a query that matches on the runtime field. This query returns the same results as the search query previously defined for the `http.response` runtime field in your index mappings, but only in the context of this specific search:

```console
GET my-index/_search
{
  "runtime_mappings": {
    "http.response": {
      "type": "long",
      "script": """
        String response=dissect('%{clientip} %{ident} %{auth} [%{@timestamp}] "%{verb} %{request} HTTP/%{httpversion}" %{response} %{size}').extract(doc["message"].value)?.response;
        if (response != null) emit(Integer.parseInt(response));
      """
    }
  },
  "query": {
    "match": {
      "http.response": "304"
    }
  },
  "fields" : ["http.response"]
}
```

```console-result
{
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "my-index",
        "_id" : "D47UqXkBByC8cgZrkbOm",
        "_score" : 1.0,
        "_source" : {
          "timestamp" : "2020-04-30T14:31:22-05:00",
          "message" : "247.37.0.0 - - [30/Apr/2020:14:31:22 -0500] \"GET /images/hm_nbg.jpg HTTP/1.0\" 304 0"
        },
        "fields" : {
          "http.response" : [
            304
          ]
        }
      }
    ]
  }
}
```


