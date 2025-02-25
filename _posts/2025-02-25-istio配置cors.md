---
layout: article
title: istio cors config
tags: k8s kubernetes istio cors
key: 2025-02-19-istio-configure-cors
---

## 参考

- <https://istio.io/latest/docs/reference/config/networking/virtual-service/#CorsPolicy>
- <https://istio.io/latest/docs/reference/config/networking/virtual-service/#StringMatch>
- <https://stackoverflow.com/questions/65862613/istio-request-authentication-getting-cors-with-result-404>

## 配置位置

可以在gateway配置，也可以在virtual service中配置

## 配置内容

```yaml
spec:
  gateways:
  - xx-ingress-gateway
  hosts:
  - xx.xx.xx.xx
  http:
  - match:
    - uri:
        prefix: /api/v1/user
    rewrite:
      uri: /svc/user
    route:
    - destination:
      host: xx.xx.xx.xx
    corsPolicy:
      allowCredentials: true # 是否允许调用方根据凭据发送实际请求，而非preflight请求(即OPTIONS请求)
      allowHeaders:
      - authorization
      - content-type
      - accept
      - accept-language
      - cache-control
      - pragma
      allowMethods:
      - OPTIONS
      - HEAD
      - GET
      allowOrigins:
      - exact: http://xx.xx.xx.xx:xx # 精确匹配
      - exact: http://xx.xx.xx.xx:xx
      - regex: http://192\.168\.160\.11:.* # 正则匹配
      exposeHeaders:
      - content-length
      - content-type
      maxAge: 1h
```

## 额外配置

如果使用了istio external server的话，可能需要额外配置。原因是客户端发起cors请求时，会先发`OPTIONS`请求，并且Header里是不会带token等内容的。如果istio external server里需要校验token的话，就会被拦截，因此需要将OPTIONS请求放开。  

```yaml
# 方法1
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: xxx
  namespace: istio-system
spec:
  action: ALLOW
  rules:
  # 在ALLOW的定义下，直接放开所有的OPTIONS请求
  - to:
    - operation:
        methods:
        - OPTIONS
```

```yaml
# 方法2
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: xxx
  namespace: istio-system
spec:
  action: CUSTOM
  provider:
    name: "xx-provider"
  rules:
  - to:
    - operation:
        paths: ["/api/v1/*"]
        # 在COUSTOM的定义下，在指定的methods中剔除OPTIONS
        methods:
        - HEAD
        - GET
        - POST
        - PUT
        - DELETE
```
