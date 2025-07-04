---
navigation_title: PagerDuty action
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/actions-pagerduty.html
applies_to:
  stack: ga
products:
  - id: elasticsearch
---

# PagerDuty action [actions-pagerduty]

Use the PagerDuty action to create events in [ PagerDuty](https://pagerduty.com/). To create PagerDuty events, you must [configure at least one PagerDuty account](#configuring-pagerduty) in `elasticsearch.yml`.

## Configuring PagerDuty actions [configuring-pagerduty-actions]

You configure PagerDuty actions in the `actions` array. Action-specific attributes are specified using the `pagerduty` keyword.

The following snippet shows a simple PagerDuty action definition:

```json
"actions" : {
  "notify-pagerduty" : {
    "transform" : { ... },
    "throttle_period" : "5m",
    "pagerduty" : {
      "description" : "Main system down, please check!" <1>
    }
  }
}
```

1. Description of the message

## Adding meta information to a PagerDuty incident [adding-context-and-payloads-to-pagerduty-actions]

To give the PagerDuty incident some more context, you can attach the payload as well as an array of contexts to the action.

```json
"actions" : {
  "notify-pagerduty" : {
    "throttle_period" : "5m",
    "pagerduty" : {
      "account" : "team1",
      "description" : "Main system down, please check! Happened at {{ctx.execution_time}}",
      "attach_payload" : true,
      "client" : "/foo/bar/{{ctx.watch_id}}",
      "client_url" : "http://www.example.org/",
      "contexts" : [
        {
          "type": "link",
          "href": "http://acme.pagerduty.com"
        },{
          "type": "link",
          "href": "http://acme.pagerduty.com",
          "text": "View the incident on {{ctx.payload.link}}"
        }
      ]
    }
  }
}
```

## Pagerduty action attributes [pagerduty-action-attributes]

| Name | Required | Description |
| --- | --- | --- |
| `account` | no | The account to use, falls back to the default one.                            The account needs a `service_api_key` attribute. |

::::{note}
Although some of the attributes below have names which match the PagerDuty "Events API v1" parameter names, the "Events API v2" API is finally used by translating the attributes appropriately.
::::

$$$pagerduty-event-trigger-incident-attributes$$$

| Name | Required | Description |
| --- | --- | --- |
| `description` | yes | A quick description for this event |
| `event_type` | no | The event type to sent. Must be one of `trigger`,                                `resolve` or `acknowledge`. Defaults to `trigger`. |
| `incident_key` | no | The incident key on the pagerduty side, also used                                for de-duplication and allows to resolve or acknowledge                                incidents. |
| `client` | no | Name of the client triggering the incident, i.e.                                `Watcher Monitoring` |
| `client_url` | no | A client URL to visit to get more detailed information. |
| `attach_payload` | no | If set to `true` the payload is attached as a detail                                to the API call. Defaults to `false`. |
| `contexts` | no | An array of objects, that allow you to provide                                additional links or images in order to provide more                                context to the trigger. |
| `proxy.host` | no | The proxy host to use (only in combination with `proxy.port`) |
| `proxy.port` | no | The proxy port to use (only in combination with `proxy.host`) |

You can configure defaults for the above values for the whole service using the `xpack.notification.pagerduty.event_defaults.*` properties as well as per account using `xpack.notification.pagerduty.account.your_account_name.event_defaults.*`

::::{note}
All of those objects have templating support, so you can use data from the context and the payload as part of all the fields.
::::

$$$pagerduty-event-trigger-context-attributes$$$

| Name | Required | Description |
| --- | --- | --- |
| `type` | yes | One of `link` or `image`. |
| `href` | yes/no | A link to include more information. Must be there if the                      type is `link`, optional if the type is `image` |
| `src` | no | A src attribute for the `image` type. |

## Configuring PagerDuty accounts [configuring-pagerduty]

You configure the accounts {{watcher}} uses to communicate with PagerDuty in the `xpack.notification.pagerduty` namespace in [`elasticsearch.yml`](/deploy-manage/stack-settings.md).

To configure a PagerDuty account, you need the API integration key for the PagerDuty service you want to send notifications to. To get the key:

1. Log in to [pagerduty.com](http://pagerduty.com) as an account administrator.
2. Go to **Services** and select the PagerDuty service you wish to target.
3. Click the **Integrations** tab and add an **Events API V2** integration if one does not already exist.
4. Copy the API integration key for use below.

To configure a PagerDuty account in the keystore, you must specify an account name and integration key, (see [Secure settings](../../../deploy-manage/security/secure-settings.md)):

```yaml
bin/elasticsearch-keystore add xpack.notification.pagerduty.account.my_pagerduty_account.secure_service_api_key
```

::::{admonition} Deprecated in 7.0.0.
:class: warning

Storing the service api key in the YAML file or via cluster update settings is still supported, but the keystore setting should be used.
::::

You can also specify defaults for the [PagerDuty event attributes](#pagerduty-event-trigger-incident-attributes):

```yaml
xpack.notification.pagerduty:
  account:
    my_pagerduty_account:
      event_defaults:
        description: "Watch notification"
        incident_key: "my_incident_key"
        client: "my_client"
        client_url: http://www.example.org
        event_type: trigger
        attach_payload: true
```

If you configure multiple PagerDuty accounts, you either need to set a default account or specify which account the event should be sent with in the `pagerduty` action.

```yaml
xpack.notification.pagerduty:
  default_account: team1
  account:
    team1:
      ...
    team2:
      ...
```
