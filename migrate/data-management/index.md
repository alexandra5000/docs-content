---
navigation_title: Modernize data management
mapped_pages:
  - https://www.elastic.co/guide/en/cloud-enterprise/current/ece-migrate-index-management.html
  - https://www.elastic.co/guide/en/cloud/current/ec-migrate-index-management.html
applies_to:
  stack: ga
products:
  - id: elasticsearch
  - id: cloud-enterprise
  - id: cloud-hosted
---

# Modernize data management [migrate-ilm]

If you're using older lifecycle or allocation patterns, these guides help you migrate to current {{es}} features:

[](/migrate/data-management/to-ilm.md) {applies_to}`ess:` {applies_to}`ece:`
:   Migrate {{es}} indices in an {{ech}} or {{ece}} deployment from deprecated index curation or another periodic management mechanism to {{ilm}}.

[](/manage-data/lifecycle/index-lifecycle-management/manage-existing-indices.md)  {applies_to}`self:` {applies_to}`eck:`
:   Migrate {{es}} indices in a self-managed environment from deprecated index curation, or any other lifecycle mechanism, to {{ilm}}.

[](/migrate/data-management/to-data-tiers.md) {applies_to}`stack:`
:   Migrate from custom node attributes and attribute-based allocation filters to built-in node roles, so that {{ilm-init}} can automatically move indices between data tiers.

[](/migrate/data-management/ilm-to-data-stream-lifecycle.md) {applies_to}`stack:`
:   Migrate a data stream from {{ilm-init}} to the newer, simpler data stream lifecycle.

[](/migrate/data-management/rollup-to-downsampling.md) {applies_to}`stack:`
:   Migrate from the deprecated rollup feature to downsampling for time series data compaction.
