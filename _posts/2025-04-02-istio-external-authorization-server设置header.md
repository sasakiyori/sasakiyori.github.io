---
layout: article
title: istio external authorization server设置header
tags: k8s kubernetes istio authorization envoy
key: 2025-02-19-istio-external-authorization-server-set-header
---

## 背景

在通过istio external authorization server鉴权的过程后，我们可能想额外往用户请求里设置一些header，以便于真正的服务去解析。

例如通过特殊header签名，交换到Authorization token，然后将这个token设置到request header上，从而让真正的服务只需要关心Authorization这个header即可，而不需要关心其他的header。

## 做法

```go
import (
    corev3 "github.com/envoyproxy/go-control-plane/envoy/config/core/v3"
    authv3 "github.com/envoyproxy/go-control-plane/envoy/service/auth/v3"
    "google.golang.org/genproto/googleapis/rpc/status"
    "google.golang.org/grpc/codes"
)

func (s *MyExtGrpcServer) Check(ctx context.Context, request *authv3.CheckRequest) (*authv3.CheckResponse, error) {
    return &authv3.CheckResponse{
        Status: &status.Status{
            Code: int32(codes.OK),
        },
        HttpResponse: &authv3.CheckResponse_OkResponse{
            OkResponse: &authv3.OkHttpResponse{
                // 在转发给实际客户端的请求头中，根据AppendAction修改或添加request header
                Headers: []*corev3.HeaderValueOption{
                    {
                        // 这里试过自定义key 对于grpc ext server来说无效
                        // http ext server可能有效，未测试。可能需要在istio mesh的配置中，添加headersToDownstreamOnAllow的配置
                        // 详情见参考文档
                        Header: &corev3.HeaderValue{Key: "Authorization", Value: "Bearer xxx"},
                        AppendAction: corev3.HeaderValueOption_OVERWRITE_IF_EXISTS_OR_ADD,
                    },
                },
                // 在实际客户端返回结果时，根据AppendAction添加或修改response header
                ResponseHeadersToAdd: []*corev3.HeaderValueOption{
                    {
                        Header: &corev3.HeaderValue{Key: "Set-Cookie", Value: "Authorization=Bearer xxx; Max-Age=3600; Path=/; HttpOnly"},
                        AppendAction: corev3.HeaderValueOption_OVERWRITE_IF_EXISTS_OR_ADD,
                    },
                },
            },
        },
    }, nil
}
```

## 参考

- <https://istio.io/latest/docs/tasks/security/authorization/authz-custom/#define-the-external-authorizer>
- <https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-EnvoyExternalAuthorizationGrpcProvider>
- <https://github.com/envoyproxy/go-control-plane/blob/main/envoy/service/auth/v3/external_auth.pb.go#L162>
- <https://github.com/envoyproxy/go-control-plane/blob/main/envoy/service/auth/v3/external_auth.pb.go#L189>
