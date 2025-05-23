---
navigation_title: Remote clusters
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/remote-clusters-troubleshooting.html
applies_to:
  stack:
  deployment:
    eck:
    ess:
    ece:
    self:
products:
  - id: elasticsearch
---



# Troubleshoot remote clusters [remote-clusters-troubleshooting]


You may encounter several issues when setting up a remote cluster for {{ccr}} or {{ccs}}.

## General troubleshooting [remote-clusters-troubleshooting-general]

### Checking whether a remote cluster has connected successfully [remote-clusters-troubleshooting-check-connection]

A successful call to the cluster settings update API for adding or updating remote clusters does not necessarily mean the configuration is successful. Use the [remote cluster info API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-remote-info) to verify that a local cluster is successfully connected to a remote cluster.

```console
GET /_remote/info
```

The API should return `"connected" : true`. When using [API key authentication](../../deploy-manage/remote-clusters/remote-clusters-api-key.md), it should also return `"cluster_credentials": "::es_redacted::"`.

```console-result
{
  "cluster_one" : {
    "seeds" : [
      "127.0.0.1:9443"
    ],
    "connected" : true, <1>
    "num_nodes_connected" : 1,
    "max_connections_per_cluster" : 3,
    "initial_connect_timeout" : "30s",
    "skip_unavailable" : false,
    "cluster_credentials": "::es_redacted::", <2>
    "mode" : "sniff"
  }
}
```

1. The remote cluster has connected successfully.
2. If present, indicates the remote cluster has connected using [API key authentication](../../deploy-manage/remote-clusters/remote-clusters-api-key.md) instead of [certificate based authentication](../../deploy-manage/remote-clusters/remote-clusters-cert.md).



### Enabling the remote cluster server [remote-clusters-troubleshooting-enable-server]

When using API key authentication, cross-cluster traffic happens on the remote cluster interface, instead of the transport interface. The remote cluster interface is not enabled by default. This means a node is not ready to accept incoming cross-cluster requests by default, while it is ready to send outgoing cross-cluster requests. Ensure you’ve enabled the remote cluster server on every node of the remote cluster. In [`elasticsearch.yml`](/deploy-manage/stack-settings.md):

* Set [`remote_cluster_server.enabled`](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#remote-cluster-network-settings) to `true`.
* Configure the bind and publish address for remote cluster server traffic, for example using [`remote_cluster.host`](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#remote-cluster-network-settings). Without configuring the address, remote cluster traffic may be bound to the local interface, and remote clusters running on other machines can’t connect.
* Optionally, configure the remote server port using [`remote_cluster.port`](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#remote_cluster.port) (defaults to `9443`).



## Common issues [remote-clusters-troubleshooting-common-issues]

The following issues are listed in the order they may occur while setting up a remote cluster.

### Remote cluster not reachable [remote-clusters-not-reachable]

#### Symptom [_symptom]

A local cluster may not be able to reach a remote cluster for many reasons. For example, the remote cluster server may not be enabled, an incorrect host or port may be configured, or a firewall may be blocking traffic. When a remote cluster is not reachable, check the logs of the local cluster for a `connect_exception`.

When the remote cluster is configured using proxy mode:

```txt
[2023-06-28T16:36:47,264][WARN ][o.e.t.ProxyConnectionStrategy] [local-node] failed to open any proxy connections to cluster [my]
org.elasticsearch.transport.ConnectTransportException: [][192.168.0.42:9443] **connect_exception**
```

When the remote cluster is configured using sniff mode:

```txt
[2023-06-28T16:38:37,731][WARN ][o.e.t.SniffConnectionStrategy] [local-node] fetching nodes from external cluster [my] failed
org.elasticsearch.transport.ConnectTransportException: [][192.168.0.42:9443] **connect_exception**
```


#### Resolution [_resolution]

* Check the host and port for the remote cluster are correct.
* Ensure the [remote cluster server is enabled](#remote-clusters-troubleshooting-enable-server) on the remote cluster.
* Ensure no firewall is blocking the communication.



### Remote cluster connection is unreliable [remote-clusters-unreliable-network]

#### Symptom [_symptom_2]

The local cluster can connect to the remote cluster, but the connection does not work reliably. For example, some cross-cluster requests may succeed while others report connection errors, time out, or appear to be stuck waiting for the remote cluster to respond.

When {{es}} detects that the remote cluster connection is not working, it will report the following message in its logs:

```txt
[2023-06-28T16:36:47,264][INFO ][o.e.t.ClusterConnectionManager] [local-node] transport connection to [{my-remote#192.168.0.42:9443}{...}] closed by remote
```

This message will also be logged if the node of the remote cluster to which {{es}} is connected is shut down or restarted.

Note that with some network configurations it could take minutes or hours for the operating system to detect that a connection has stopped working. Until the failure is detected and reported to {{es}}, requests involving the remote cluster may time out or may appear to be stuck.


#### Resolution [_resolution_2]

* Ensure that the network between the clusters is as reliable as possible.
* Ensure that the network is configured to permit [Long-lived idle connections](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#long-lived-connections).
* Ensure that the network is configured to detect faulty connections quickly. In particular, you must enable and fully support TCP keepalives, and set a short [retransmission timeout](../../deploy-manage/deploy/self-managed/system-config-tcpretries.md).
* On Linux systems, execute `ss -tonie` to verify the details of the configuration of each network connection between the clusters.
* If the problems persist, capture network packets at both ends of the connection and analyse the traffic to look for delays and lost messages.



### TLS trust not established [remote-clusters-troubleshooting-tls-trust]

TLS can be misconfigured on the local or the remote cluster. The result is that the local cluster does not trust the certificate presented by the remote cluster.

#### Symptom [_symptom_3]

The local cluster logs `failed to establish trust with server`:

```txt
[2023-06-29T09:40:55,465][WARN ][o.e.c.s.DiagnosticTrustManager] [local-node] **failed to establish trust with server** at [192.168.0.42]; the server provided a certificate with subject name [CN=remote_cluster], fingerprint [529de35e15666ffaa26afa50876a2a48119db03a], no keyUsage and no extendedKeyUsage; the certificate is valid between [2023-01-29T12:08:37Z] and [2032-08-29T12:08:37Z] (current time is [2023-08-16T23:40:55.464275Z], certificate dates are valid); the session uses cipher suite [TLS_AES_256_GCM_SHA384] and protocol [TLSv1.3]; the certificate has subject alternative names [DNS:localhost,DNS:localhost6.localdomain6,IP:127.0.0.1,IP:0:0:0:0:0:0:0:1,DNS:localhost4,DNS:localhost6,DNS:localhost.localdomain,DNS:localhost4.localdomain4,IP:192.168.0.42]; the certificate is issued by [CN=Elastic Auto RemoteCluster CA] but the server did not provide a copy of the issuing certificate in the certificate chain; this ssl context ([(shared) (with trust configuration: JDK-trusted-certs)]) is not configured to trust that issuer but trusts [97] other issuers
sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

The remote cluster logs `client did not trust this server's certificate`:

```txt
[2023-06-29T09:40:55,478][WARN ][o.e.x.c.s.t.n.SecurityNetty4Transport] [remote-node] **client did not trust this server's certificate**, closing connection Netty4TcpChannel{localAddress=/192.168.0.42:9443, remoteAddress=/192.168.0.84:57305, profile=_remote_cluster}
```


#### Resolution [_resolution_3]

Read the warn log message on the local cluster carefully to determine the exact cause of the failure. For example:

* Is the remote cluster certificate not signed by a trusted CA? This is the most likely cause.
* Is hostname verification failing?
* Is the certificate expired?

Once you know the cause, you should be able to fix it by adjusting the remote cluster related SSL settings on either the local cluster or the remote cluster.

Often, the issue is on the local cluster. For example, fix it by configuring necessary trusted CAs (`xpack.security.remote_cluster_client.ssl.certificate_authorities`).

If you change the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) file, the associated cluster needs to be restarted for the changes to take effect.




## API key authentication issues [remote-clusters-troubleshooting-api-key]

### Connecting to transport port when using API key authentication [remote-clusters-troubleshooting-transport-port-api-key]

When using API key authentication, a local cluster should connect to a remote cluster’s remote cluster server port (defaults to `9443`) instead of the transport port (defaults to `9300`). A misconfiguration can lead to a number of symptoms:

#### Symptom 1 [_symptom_1]

It’s recommended to use different CAs and certificates for the transport interface and the remote cluster server interface. If this recommendation is followed, a remote cluster client node does not trust the server certificate presented by a remote cluster on the transport interface.

The local cluster logs `failed to establish trust with server`:

```txt
[2023-06-28T12:48:46,575][WARN ][o.e.c.s.DiagnosticTrustManager] [local-node] **failed to establish trust with server** at [1192.168.0.42]; the server provided a certificate with subject name [CN=transport], fingerprint [c43e628be2a8aaaa4092b82d78f2bc206c492322], no keyUsage and no extendedKeyUsage; the certificate is valid between [2023-01-29T12:05:53Z] and [2032-08-29T12:05:53Z] (current time is [2023-06-28T02:48:46.574738Z], certificate dates are valid); the session uses cipher suite [TLS_AES_256_GCM_SHA384] and protocol [TLSv1.3]; the certificate has subject alternative names [DNS:localhost,DNS:localhost6.localdomain6,IP:127.0.0.1,IP:0:0:0:0:0:0:0:1,DNS:localhost4,DNS:localhost6,DNS:localhost.localdomain,DNS:localhost4.localdomain4,IP:192.168.0.42]; the certificate is issued by [CN=Elastic Auto Transport CA] but the server did not provide a copy of the issuing certificate in the certificate chain; this ssl context ([xpack.security.remote_cluster_client.ssl (with trust configuration: PEM-trust{/rcs2/ssl/remote-cluster-ca.crt})]) is not configured to trust that issuer, it only trusts the issuer [CN=Elastic Auto RemoteCluster CA] with fingerprint [ba2350661f66e46c746c1629f0c4b645a2587ff4]
sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

The remote cluster logs `client did not trust this server's certificate`:

```txt
[2023-06-28T12:48:46,584][WARN ][o.e.x.c.s.t.n.SecurityNetty4Transport] [remote-node] **client did not trust this server's certificate**, closing connection Netty4TcpChannel{localAddress=/192.168.0.42:9309, remoteAddress=/192.168.0.84:60810, profile=default}
```


#### Symptom 2 [_symptom_2_2]

The CA and certificate can be shared between the transport and remote cluster server interface. Since a remote cluster client does not have a client certificate by default, the server will fail to verify the client certificate.

The local cluster logs `Received fatal alert: bad_certificate`:

```txt
[2023-06-28T12:43:30,705][WARN ][o.e.t.TcpTransport       ] [local-node] exception caught on transport layer [Netty4TcpChannel{localAddress=/192.168.0.84:60738, remoteAddress=/192.168.0.42:9309, profile=_remote_cluster}], closing connection
io.netty.handler.codec.DecoderException: javax.net.ssl.SSLHandshakeException: **Received fatal alert: bad_certificate**
```

The remote cluster logs `Empty client certificate chain`:

```txt
[2023-06-28T12:43:30,772][WARN ][o.e.t.TcpTransport       ] [remote-node] exception caught on transport layer [Netty4TcpChannel{localAddress=/192.168.0.42:9309, remoteAddress=/192.168.0.84:60783, profile=default}], closing connection
io.netty.handler.codec.DecoderException: javax.net.ssl.SSLHandshakeException: **Empty client certificate chain**
```


#### Symptom 3 [_symptom_3_2]

If the remote cluster client is configured for mTLS and provides a valid client certificate, the connection fails because the client does not send the expected authentication header.

The local cluster logs `missing authentication`:

```txt
[2023-06-28T13:04:52,710][WARN ][o.e.t.ProxyConnectionStrategy] [local-node] failed to open any proxy connections to cluster [my]
org.elasticsearch.transport.RemoteTransportException: [remote-node][192.168.0.42:9309][cluster:internal/remote_cluster/handshake]
Caused by: org.elasticsearch.ElasticsearchSecurityException: **missing authentication** credentials for action [cluster:internal/remote_cluster/handshake]
```

This does not show up in the logs of the remote cluster.


#### Symptom 4 [_symptom_4]

If anonymous access is enabled on the remote cluster and it does not require authentication, depending on the privileges of the anonymous user, the local cluster may log the following.

If the anonymous user does not the have necessary privileges to make a connection, the local cluster logs `unauthorized`:

```txt
org.elasticsearch.transport.RemoteTransportException: [remote-node][192.168.0.42:9309][cluster:internal/remote_cluster/handshake]
Caused by: org.elasticsearch.ElasticsearchSecurityException: action [cluster:internal/remote_cluster/handshake] is **unauthorized** for user [anonymous_foo] with effective roles [reporting_user], this action is granted by the cluster privileges [cross_cluster_search,cross_cluster_replication,manage,all]
```

If the anonymous user has necessary privileges, for example it is a superuser, the local cluster logs `requires channel profile to be [_remote_cluster], but got [default]`:

```txt
[2023-06-28T13:09:52,031][WARN ][o.e.t.ProxyConnectionStrategy] [local-node] failed to open any proxy connections to cluster [my]
org.elasticsearch.transport.RemoteTransportException: [remote-node][192.168.0.42:9309][cluster:internal/remote_cluster/handshake]
Caused by: java.lang.IllegalArgumentException: remote cluster handshake action **requires channel profile to be [_remote_cluster], but got [default]**
```


#### Resolution [_resolution_4]

Check the port number and ensure you are indeed connecting to the remote cluster server instead of the transport interface.



### Connecting without a cross-cluster API key [remote-clusters-troubleshooting-no-api-key]

A local cluster uses the presence of a cross-cluster API key to determine the model with which it connects to a remote cluster. If a cross-cluster API key is present, it uses API key based authentication. Otherwise, it uses certificate based authentication. You can check what model is being used with the [remote cluster info API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-remote-info) on the local cluster:

```console
GET /_remote/info
```

The API should return `"connected" : true`. When using [API key authentication](../../deploy-manage/remote-clusters/remote-clusters-api-key.md), it should also return `"cluster_credentials": "::es_redacted::"`.

```console-result
{
  "cluster_one" : {
    "seeds" : [
      "127.0.0.1:9443"
    ],
    "connected" : true, <1>
    "num_nodes_connected" : 1,
    "max_connections_per_cluster" : 3,
    "initial_connect_timeout" : "30s",
    "skip_unavailable" : false,
    "cluster_credentials": "::es_redacted::", <2>
    "mode" : "sniff"
  }
}
```

1. The remote cluster has connected successfully.
2. If present, indicates the remote cluster has connected using [API key authentication](../../deploy-manage/remote-clusters/remote-clusters-api-key.md) instead of [certificate based authentication](../../deploy-manage/remote-clusters/remote-clusters-cert.md).


Besides checking the response of the remote cluster info API, you can also check the logs.

#### Symptom 1 [_symptom_1_2]

If no cross-cluster API key is used, the local cluster uses the certificate based authentication method, and connects to the remote cluster using the TLS configuration of the transport interface. If the remote cluster has different TLS CA and certificate for transport and remote cluster server interfaces (which is the recommendation), TLS verification will fail.

The local cluster logs `failed to establish trust with server`:

```txt
[2023-06-28T12:51:06,452][WARN ][o.e.c.s.DiagnosticTrustManager] [local-node] **failed to establish trust with server** at [<unknown host>]; the server provided a certificate with subject name [CN=remote_cluster], fingerprint [529de35e15666ffaa26afa50876a2a48119db03a], no keyUsage and no extendedKeyUsage; the certificate is valid between [2023-01-29T12:08:37Z] and [2032-08-29T12:08:37Z] (current time is [2023-06-28T02:51:06.451581Z], certificate dates are valid); the session uses cipher suite [TLS_AES_256_GCM_SHA384] and protocol [TLSv1.3]; the certificate has subject alternative names [DNS:localhost,DNS:localhost6.localdomain6,IP:127.0.0.1,IP:0:0:0:0:0:0:0:1,DNS:localhost4,DNS:localhost6,DNS:localhost.localdomain,DNS:localhost4.localdomain4,IP:192.168.0.42]; the certificate is issued by [CN=Elastic Auto RemoteCluster CA] but the server did not provide a copy of the issuing certificate in the certificate chain; this ssl context ([xpack.security.transport.ssl (with trust configuration: PEM-trust{/rcs2/ssl/transport-ca.crt})]) is not configured to trust that issuer, it only trusts the issuer [CN=Elastic Auto Transport CA] with fingerprint [bbe49e3f986506008a70ab651b188c70df104812]
sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

The remote cluster logs `client did not trust this server's certificate`:

```txt
[2023-06-28T12:52:16,914][WARN ][o.e.x.c.s.t.n.SecurityNetty4Transport] [remote-node] **client did not trust this server's certificate**, closing connection Netty4TcpChannel{localAddress=/192.168.0.42:9443, remoteAddress=/192.168.0.84:60981, profile=_remote_cluster}
```


#### Symptom 2 [_symptom_2_3]

Even if TLS verification is not an issue, the connection fails due to missing credentials.

The local cluster logs `Make sure you have configured remote cluster credentials`:

```txt
Caused by: java.lang.IllegalArgumentException: Cross cluster requests through the dedicated remote cluster server port require transport header [_cross_cluster_access_credentials] but none found. **Make sure you have configured remote cluster credentials** on the cluster originating the request.
```

This does not show up in the logs of the remote cluster.


#### Resolution [_resolution_5]

Add the cross-cluster API key to {{es}} keystore on every node of the local cluster. Use the [Nodes reload secure settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-reload-secure-settings) API to reload the keystore.



### Using the wrong API key type [remote-clusters-troubleshooting-wrong-api-key-type]

API key based authentication requires [cross-cluster API keys](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-cross-cluster-api-key). It does not work with [REST API keys](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-api-key).

#### Symptom [_symptom_5]

The local cluster logs `authentication expected API key type of [cross_cluster]`:

```txt
[2023-06-28T13:26:53,962][WARN ][o.e.t.ProxyConnectionStrategy] [local-node] failed to open any proxy connections to cluster [my]
org.elasticsearch.transport.RemoteTransportException: [remote-node][192.168.0.42:9443][cluster:internal/remote_cluster/handshake]
Caused by: org.elasticsearch.ElasticsearchSecurityException: **authentication expected API key type of [cross_cluster]**, but API key [agZXJocBmA2beJfq2yKu] has type [rest]
```

This does not show up in the logs of the remote cluster.


#### Resolution [_resolution_6]

Ask the remote cluster administrator to create and distribute a [cross-cluster API key](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-cross-cluster-api-key). Replace the existing API key in the {{es}} keystore with this cross-cluster API key on every node of the local cluster. Use the [Nodes reload secure settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-reload-secure-settings) API to reload the keystore.



### Invalid API key [remote-clusters-troubleshooting-non-valid-api-key]

A cross-cluster API can fail to authenticate. For example, when its credentials are incorrect, or if it’s invalidated or expired.

#### Symptom [_symptom_6]

The local cluster logs `unable to authenticate`:

```txt
[2023-06-28T13:22:58,264][WARN ][o.e.t.ProxyConnectionStrategy] [local-node] failed to open any proxy connections to cluster [my]
org.elasticsearch.transport.RemoteTransportException: [remote-node][192.168.0.42:9443][cluster:internal/remote_cluster/handshake]
Caused by: org.elasticsearch.ElasticsearchSecurityException: **unable to authenticate** user [agZXJocBmA2beJfq2yKu] for action [cluster:internal/remote_cluster/handshake]
```

The remote cluster logs `Authentication using apikey failed`:

```txt
[2023-06-28T13:24:38,744][WARN ][o.e.x.s.a.ApiKeyAuthenticator] [remote-node] **Authentication using apikey failed** - invalid credentials for API key [agZXJocBmA2beJfq2yKu]
```


#### Resolution [_resolution_7]

Ask the remote cluster administrator to create and distribute a [cross-cluster API key](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-cross-cluster-api-key). Replace the existing API key in the {{es}} keystore with this cross-cluster API key on every node of the local cluster. Use the [Nodes reload secure settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-reload-secure-settings) API to reload the keystore.



### API key or local user has insufficient privileges [remote-clusters-troubleshooting-insufficient-privileges]

The effective permission for a local user running requests on a remote cluster is determined by the intersection of the cross-cluster API key’s privileges and the local user’s `remote_indices` privileges.

#### Symptom [_symptom_7]

Request failures due to insufficient privileges result in API responses like:

```js
{
    "type": "security_exception",
    "reason": "action [indices:data/read/search] towards remote cluster is **unauthorized for user** [foo] with assigned roles [foo-role] authenticated by API key id [agZXJocBmA2beJfq2yKu] of user [elastic-admin] on indices [cd], this action is granted by the index privileges [read,all]"
}
```

This does not show up in any logs.


#### Resolution [_resolution_8]

1. Check that the local user has the necessary `remote_indices` or `remote_cluster` privileges. Grant sufficient `remote_indices` or `remote_cluster` privileges if necessary.
2. If permission is not an issue locally, ask the remote cluster administrator to create and distribute a [cross-cluster API key](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-cross-cluster-api-key). Replace the existing API key in the {{es}} keystore with this cross-cluster API key on every node of the local cluster. Use the [Nodes reload secure settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-reload-secure-settings) API to reload the keystore.



### Local user has no `remote_indices` privileges [remote-clusters-troubleshooting-no-remote_indices-privileges]

This is a special case of insufficient privileges. In this case, the local user has no `remote_indices` privileges at all for the target remote cluster. {{es}} can detect that and issue a more explicit error response.

#### Symptom [_symptom_8]

This results in API responses like:

```js
{
    "type": "security_exception",
    "reason": "action [indices:data/read/search] towards remote cluster [my] is unauthorized for user [foo] with effective roles [] (assigned roles [foo-role] were not found) because **no remote indices privileges apply for the target cluster**"
}
```


#### Resolution [_resolution_9]

Grant sufficient `remote_indices` privileges to the local user.




