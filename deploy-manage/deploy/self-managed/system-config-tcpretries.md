---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config-tcpretries.html
applies_to:
  deployment:
    self:
products:
  - id: elasticsearch
---

# Decrease the TCP retransmission timeout [system-config-tcpretries]

::::{note} 
This is only relevant for Linux.
::::

Each pair of {{es}} nodes communicates via a number of TCP connections which [remain open](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#long-lived-connections) until one of the nodes shuts down or communication between the nodes is disrupted by a failure in the underlying infrastructure.

TCP provides reliable communication over occasionally unreliable networks by hiding temporary network disruptions from the communicating applications. Your operating system will retransmit any lost messages a number of times before informing the sender of any problem. {{es}} must wait while the retransmissions are happening and can only react once the operating system decides to give up. Users must therefore also wait for a sequence of retransmissions to complete.

Most Linux distributions default to retransmitting any lost packets 15 times. Retransmissions back off exponentially, so these 15 retransmissions take over 900 seconds to complete. This means it takes Linux many minutes to detect a network partition or a failed node with this method. Windows defaults to just 5 retransmissions which corresponds with a timeout of around 13 seconds.

The Linux default allows for communication over networks that may experience very long periods of packet loss, but this default is excessive and even harmful on the high quality networks used by most {{es}} installations. When a cluster detects a node failure it reacts by reallocating lost shards, rerouting searches, and maybe electing a new master node. Highly available clusters must be able to detect node failures promptly, which can be achieved by reducing the permitted number of retransmissions. Connections to [remote clusters](../../remote-clusters.md) should also prefer to detect failures much more quickly than the Linux default allows. Linux users should therefore reduce the maximum number of TCP retransmissions.

You can decrease the maximum number of TCP retransmissions to `5` by running the following command as `root`. Five retransmissions corresponds with a timeout of around 13 seconds.

```sh
sysctl -w net.ipv4.tcp_retries2=5
```

To set this value permanently, update the `net.ipv4.tcp_retries2` setting in `/etc/sysctl.conf`. To verify after rebooting, run `sysctl net.ipv4.tcp_retries2`.

::::{important}
This setting applies to all TCP connections and will affect the reliability of communication with systems other than {{es}} clusters too. If your clusters communicate with external systems over a low quality network then you may need to select a higher value for `net.ipv4.tcp_retries2`. For this reason, {{es}} does not adjust this setting automatically.
::::


## Related configuration [_related_configuration]

{{es}} also implements its own internal health checks with timeouts that are much shorter than the default retransmission timeout on Linux. Since these are application-level health checks their timeouts must allow for application-level effects such as garbage collection pauses. You should not reduce any timeouts related to these application-level health checks.

You must also ensure your network infrastructure does not interfere with the long-lived connections between nodes, [even if those connections appear to be idle](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#long-lived-connections). Devices which drop connections when they reach a certain age are a common source of problems to {{es}} clusters, and must not be used.


