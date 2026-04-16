---
navigation_title: Migrate APM agents via the Collector
applies_to:
  stack: ga 9.2
  serverless:
    observability:
products:
  - id: observability
  - id: edot-collector
---

# Migrate classic APM agents via the {{edot}} Collector

If you're using classic Elastic APM agents and want to move to an OpenTelemetry-based pipeline, the {{edot}} Collector provides a migration bridge through the [Elastic APM intake receiver](elastic-agent://reference/edot-collector/components/elasticapmintakereceiver.md). This lets your existing APM agents continue sending data unchanged while you gradually re-instrument your applications with OpenTelemetry SDKs.

## How it works

The Elastic APM intake receiver implements the Elastic Intake v2 protocol, making the {{edot}} Collector behave like APM Server from the perspective of your agents. Telemetry is stored in the same format and indices as before. There is no change required to your agents during the transition.

:::{important}
Real user monitoring (RUM) intake and older Elastic APM intake protocols are not supported by this receiver.
:::

## Before you begin

- Your {{stack}} must be running version 9.2 or later.
- You must have an {{edot}} Collector deployment. Refer to the [{{edot}} Collector documentation](elastic-agent://reference/edot-collector/index.md) for setup instructions.

## Steps

1. Add the `elasticapmintake` receiver to your {{edot}} Collector configuration:

   ```yaml
   receivers:
     elasticapmintake:
       agent_config:
         enabled: false
   ```

2. Point your existing APM agents at the {{edot}} Collector endpoint instead of the APM Server URL.

3. Verify that telemetry data (traces, metrics, logs) appears correctly in the Observability UIs in {{kib}}.

4. Gradually re-instrument your applications using the language-specific [EDOT SDK migration guides](/migrate/apm-agents-to-edot/index.md) as your migration timeline allows.

## Next steps

Once your applications are re-instrumented with EDOT SDKs, you can remove the `elasticapmintake` receiver from your Collector configuration and route data directly through the standard OpenTelemetry pipeline.

For full configuration options including TLS, mTLS, and API key authentication, refer to the [Elastic APM intake receiver reference](elastic-agent://reference/edot-collector/components/elasticapmintakereceiver.md).
