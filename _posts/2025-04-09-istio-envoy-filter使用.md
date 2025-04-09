---
layout: article
title: istio envoy filter使用
tags: k8s kubernetes istio authorization envoy
key: 2025-02-19-istio-envoy-filter-usage
---

## 背景

当前istio external authorization server有一个限制，就是对同一个gateway只能应用一个ext_authz server。如果有不同的ext_authz server需要使用，要么就是得拆gateway，要么得合并功能。  

考虑到istio的authorization policy其实是对envoy filter的高级封装，考虑裸写EnvoyFilter的方式来绕过这个不合理的限制。

## 实现

envoy filter启用默认是对gateway下注册的所有路由，如果要放开某些路由，则需要添加`HTTP_ROUTE`的配置

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ext-authz-envoy-filter
  namespace: istio-system
spec:
  # 通过select找到指定的gateway
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.router"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.ext_authz
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
            transport_api_version: V3
            grpc_service:
              envoy_grpc:
                # 这里填自己的ext authz服务路径
                cluster_name: outbound|9999||my-ext-authz.default.svc.cluster.local
              timeout: 0.5s
            failure_mode_allow: true
    - applyTo: HTTP_ROUTE
      match:
        context: GATEWAY
        routeConfiguration:
          vhost:
            route:
              # 不需要走ext authz的路由名称，在virtual service中配置
              # 多个virtual service可以配置同一个名称 统一管理
              name: exclude-ext-authz-check
              action: ROUTE
      patch:
        operation: MERGE
        value:
          typed_per_filter_config:
            envoy.ext_authz:
              "@type": type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthzPerRoute
              disabled: true
```

## 参考

- <https://github.com/istio/istio/issues/34041>
- <https://istio.io/latest/docs/reference/config/networking/envoy-filter/>
- <https://istio.io/v1.3/docs/reference/config/networking/v1alpha3/envoy-filter/>
- <https://stackoverflow.com/questions/61941663/envoyfilter-to-exclude-specific-hosts>
- <https://help.aliyun.com/zh/asm/user-guide/crd-fields-for-envoy-filters>
