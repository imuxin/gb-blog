---
description: Configure a WASM filter at the Route level
---

# Enable WASM filter per route

## 提出问题

&#x20;当我们采用 typed\_per\_filter\_config 来实现 wasm per route 的时候，envoy 的日志会告诉我们 wasm filter 并不支持。

```yaml
                route:
                  cluster: web_service
                typed_per_filter_config:
                  my_wasm_plugin:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
                    config:
                      name: "my_plugin"

          http_filters:
          - name: my_wasm_plugin
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
              config:
                name: "my_wasm_plugin"
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
                    local:
                      filename: "lib/envoy_filter_http_wasm_example.wasm"
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  - name: listener_1
    address:
```

## 解决问题

为此，我们在 envoy 的 issues 中找到了好多相关的问题。其中下面这个 issue 里有提到如何解决这个问题。

{% embed url="https://github.com/envoyproxy/envoy/issues/33791" %}

> Does this make sense? [envoyproxy.io/docs/envoy/latest/intro/arch\_overview/http/http\_filters#route-based-filter-chain](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_filters#route-based-filter-chain)
>
> By that way, you can disable all wasm filters by default in the HCM and them enable specific some wasm filters on specific virtual host.
>
> Considering that the wasm filter has no per route level config, you can use a hack way to enable the filter.

```yaml
# See https://github.com/envoyproxy/envoy/issues/31482 
typed_per_filter_config:
  my_wasm_plugin:
    "@type": type.googleapis.com/envoy.config.route.v3.FilterConfig
    config: # Note this config field could not be empty because the xDS API requirement.
      "@type": type.googleapis.com/google.protobuf.Empty  # Empty as a placeholder.
    is_optional: true
```

