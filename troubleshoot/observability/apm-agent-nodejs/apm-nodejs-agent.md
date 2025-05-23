---
navigation_title: APM Node.js Agent
mapped_pages:
  - https://www.elastic.co/guide/en/apm/agent/nodejs/current/troubleshooting.html
applies_to:
  stack: all
  serverless:
    observability: all
products:
  - id: apm-agent
---

# Troubleshoot APM Node.js Agent [troubleshooting]

Is something not working as expected? Don’t worry if you can’t figure out what the problem is; we’re here to help! As a first step, ensure your app is compatible with the agent’s [supported technologies](apm-agent-nodejs://reference/supported-technologies.md).

If you’re an existing Elastic customer with a support contract, create a ticket in the [Elastic Support portal](https://support.elastic.co/customers/s/login/). Other users can post in the [APM discuss forum](https://discuss.elastic.co/c/apm).

::::{important}
**Upload your complete debug logs** to a service like [GitHub Gist](https://gist.github.com) so that we can analyze the problem. Logs should include everything from when the application starts up until the first request executes. See [Debug mode](#debug-mode) for more information.
::::



## Updating to latest agent version [use-latest-agent]

The Elastic Node.js APM Agent is updated frequently and releases are not strongly tied to other components in the Elastic Stack.  Therefore, updating to the most recently released agent version is often the recommended first troubleshooting step.

See [upgrading documentation](apm-agent-nodejs://reference/upgrading.md) for more details.


## Debug mode [debug-mode]

To capture enough information for troubleshooting, perform these steps:

1. Start your app with "trace"-level logging. This can be done by setting the environment variable `ELASTIC_APM_LOG_LEVEL=trace` or adding `logLevel: 'trace'` to the `apm.start(options)` call (see [`logLevel`](apm-agent-nodejs://reference/configuration.md#log-level) for details).
2. Disable a possible custom `logger` config, because a custom logger can result in structured log data being lost. This can be done by setting the environment variable `ELASTIC_APM_LOGGER=false`.
3. Send a few HTTP requests to some of the app endpoints and/or reproduce the issue you are seeing.
4. Wait at least 10 seconds to allow the agent to try and connect to the APM Server (controlled by [`apiRequestTime`](apm-agent-nodejs://reference/configuration.md#api-request-time)).

For example:

```bash
ELASTIC_APM_LOG_LEVEL=trace ELASTIC_APM_LOGGER=false node app.js | tee -a apm-debug.log
```

If you are capturing debugging output for Elastic support, for help on the Elastic forums, or for a GitHub issue, **upload the complete debug output** to a service like [GitHub Gist](https://gist.github.com) so that we can analyze the problem.


## Common problems [common-problems]


### No data is sent to the APM Server [no-data-sent]

If there is no data for your service in the Kibana APM app, check the log output for messages like the following.

The most common source of problems are connection issues between the agent and the APM server. Look for a log message of the form `APM Server transport error ...`. For example:

```text
APM Server transport error (ECONNREFUSED): connect ECONNREFUSED 127.0.0.1:8200
```

These may indicate an issue with the agent configuration (see [`serverUrl`](apm-agent-nodejs://reference/configuration.md#server-url), [`secretToken`](apm-agent-nodejs://reference/configuration.md#secret-token) or [`apiKey`](apm-agent-nodejs://reference/configuration.md#api-key)), a network problem between agent and server, or that the APM server is down or misconfigured (see [the APM server troubleshooting docs](/troubleshoot/observability/apm.md)).

Also look for error messages starting with `Elastic APM ...`. Some examples:

```text
Elastic APM agent disabled (`active` is false)
```

This indicates that you have likely set the `active` option or the `ELASTIC_APM_ACTIVE` environment variable to `false`. See [the `active` configuration variable docs](apm-agent-nodejs://reference/configuration.md#active).

```text
Elastic APM is incorrectly configured: serverUrl "..." contains an invalid port! (allowed: 1-65535)
```


### No metrics or trace data is sent to APM Server [missing-performance-metrics]

Errors get tracked just fine, but you don’t see any performance metrics or trace data.

Make sure that the agent is **both required and started at the very top of your main app file** (usually the `index.js`, `server.js` or `app.js` file). It’s important that the agent is started before any other modules are `require`d.  If not, the agent will not be able to hook into any modules and will not be able to measure the performance of your application. See [Start gotchas](apm-agent-nodejs://reference/starting-agent.md#start-gotchas) for a number of possible surprises in correctly starting the APM agent when used with some compilers or bundlers.


## Disable the Agent [disable-agent]

In the unlikely event the agent causes disruptions to a production application, you can disable the agent while you troubleshoot.

To disable the agent, set [`active`](apm-agent-nodejs://reference/configuration.md#active) to `false`. You’ll need to restart your application for the changes to apply.

