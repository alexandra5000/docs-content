---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
applies_to:
  deployment:
    self:
products:
  - id: elasticsearch
---

# Increase virtual memory [vm-max-map-count]

{{es}} uses a [`mmapfs`](elasticsearch://reference/elasticsearch/index-settings/store.md#mmapfs) directory by default to store its indices. The default operating system limits on mmap counts is likely to be too low, which may result in out of memory exceptions.

On Linux, you can increase the limits by running the following command as `root`:

```sh
sysctl -w vm.max_map_count=262144
```

To set this value permanently, update the `vm.max_map_count` setting in `/etc/sysctl.conf`. To verify after rebooting, run `sysctl vm.max_map_count`.

The RPM and Debian packages will configure this setting automatically. No further configuration is required.

You can find out the current mmap count of a running {{es}} process using the following command, where `$PID` is the process ID of the running {{es}} process:

```sh
wc -l /proc/$PID/maps
```

