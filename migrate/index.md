---
navigation_title: Migrate your data
applies_to:
  stack: ga
  serverless: ga
---

# Migrate to Elastic

Whether you're migrating from standalone {{beats}}, classic {{apm-agent}}s, or an external SIEM, or moving between Elastic tools such as {{agent}} and {{edot}}, this section guides you through each migration path.

## Available migration paths

[Migrate to {{agent}}](/migrate/beats-to-elastic-agent/index.md)
:   Replace {{filebeat}}, {{metricbeat}}, or {{auditbeat}} with {{agent}}, a unified agent for logs, metrics, and security data.

[Migrate {{apm-agent}}s to {{edot}}](/migrate/apm-agents-to-edot/index.md)
:   Move your application instrumentation from Elastic {{product.apm}} language agents to Elastic Distributions of OpenTelemetry (EDOT).

[Migrate to the {{edot}} Collector](/migrate/to-edot-collector/index.md)
:   Move from {{apm-server}} ingestion or a contrib OpenTelemetry Collector to the EDOT Collector.

[Migrate {{es}} data](/migrate/elasticsearch-data/index.md)
:   Move data between clusters, deployment types, or cloud environments.

[Modernize data management](/migrate/data-management/index.md)
:   Migrate from legacy lifecycle patterns — index curation, {{rollup}}, custom shard allocation — to current {{es}} features.

[Migrate to {{elastic-sec}}](/migrate/to-elastic-security/index.md)
:   Move from an external SIEM such as Splunk or QRadar to {{elastic-sec}}.
