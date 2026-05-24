---
coverY: 0
---

# Istio/Ambient 模式下 waypoint 如何加载远端 WASM 文件

本文不讨论 WasmPlugin 使用 WASM，而是通过 EnvoyFilter 的方式来更加灵活的使用 WASM。

## 提出问题

我们知道 envoy 可以通过远端下载 WASM 文件的方式来加载 WASM 程序。envoy 这里不仅需要提供 WASM 的 uri 地址，并且需要提供上游的 Cluster。这里的 cluster 我们可以理解为 outboundCluster。但在 istio ambient 模式下，我们没有直接的 CRD API 来声明 outboundCluster 和 waypoint(s) 的绑定关系的。

### Envoy WASM Filter 配置介绍

参考 envoy 的 WASM filer 配置文档，示例配置如下所示，其中高亮部分的为 WASM 文件的远端下载配置。

<pre class="language-yaml" data-expandable="true"><code class="lang-yaml">                route:
                  cluster: web_service

          http_filters:
          - name: envoy.filters.http.wasm
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
              config:
                name: "my_plugin"
                root_id: "my_root_id"
                # if your wasm filter requires custom configuration you can add
                # as follows
                configuration:
                  "@type": "type.googleapis.com/google.protobuf.StringValue"
                  value: |
                    {}
                vm_config:
                  vm_id: "my_vm_id"
                  code:
<strong>                    remote:
</strong><strong>                      http_uri:
</strong><strong>                        uri: http://wasm.hub.xyz:9527/rate_limit_filter.wasm
</strong><strong>                        cluster: "outbound|9527||wasm.hub.xyz"
</strong><strong>                        timeout: 30s
</strong><strong>                      sha256: "20776bc90c8a2d40bb1e5b6e84fa9f7e0decb0dfeb195887ad548db60f3a29aa"
</strong>          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  - name: listener_1
    address:
</code></pre>

并且从 `config.core.v3.HttpUri`的[定义](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/http_uri.proto#envoy-v3-api-msg-config-core-v3-httpuri)，可以很明确的知道 Cluster 为必填项。

> cluster
>
> ([string](https://developers.google.com/protocol-buffers/docs/proto#scalar), _REQUIRED_) A cluster is created in the Envoy “cluster\_manager” config section. This field specifies the cluster name.
>
> Example:
>
> cluster: jwks\_cluster
>
> Specify how `uri` is to be fetched. Today, this requires an explicit cluster, but in the future we may support dynamic cluster creation or inline DNS resolution. See [issue](https://github.com/envoyproxy/envoy/issues/1606).

{% embed url="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/http_uri.proto#envoy-v3-api-msg-config-core-v3-httpuri" %}

{% embed url="https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/wasm_filter" %}

{% embed url="https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/wasm/v3/wasm.proto" %}

{% embed url="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-asyncdatasource" %}

{% embed url="https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-remotedatasource" %}

***

## 解决问题

阅读理解一下 istio cds 的源码，不难发现，istio 可以通过 `MeshConfig.ExtensionProviders` 来配置 WASM hub 的地址，通过这种方式将 WASM hub 的 cds 下发给所有的 waypoint。

```go
// extraServicesForProxy returns a subset of services referred from the proxy gateways, including:
// 1. MeshConfig.ExtensionProviders
// 2. RequestAuthentication.JwtRules.JwksUri
// 3. EnvoyFilters with explicitly annotated references
func (ps *PushContext) extraServicesForProxy(proxy *Proxy, patches *MergedEnvoyFilterWrapper) (sets.Set[NamespacedHostname], sets.String) {
	...
```

{% embed url="https://github1s.com/istio/istio/blob/release-1.26/pilot/pkg/networking/core/cluster.go#L240-L252" %}

ExtensionProvider 示例如下：

<pre class="language-yaml"><code class="lang-yaml">apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
  revision: 1-26
  meshConfig:
    enableTracing: false
<strong>    extensionProviders:
</strong><strong>    - name: xyz
</strong><strong>      lightstep:
</strong><strong>        port: 9527
</strong><strong>        service: istio-system/wasm.hub.xyz
</strong>  components:
    pilot:
      k8s:
        env:
        - name: "PILOT_ENABLE_AMBIENT"
          value: "true"
</code></pre>

