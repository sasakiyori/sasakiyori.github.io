---
layout: article
title: istio external authorization server问题排查
tags: istio authorization envoy
key: 2025-02-19-istio-external-authorization-server-debug
---


## 流程

### 安装`grpcurl`

```shell
$ curl -LO https://github.com/fullstorydev/grpcurl/releases/latest/download/grpcurl_1.9.2_linux_amd64.deb
$ dpkg -i grpcurl_1.9.2_linux_amd64.deb

$ grpcurl -plaintext localhost:9999 list
envoy.service.auth.v3.Authorization
grpc.reflection.v1.ServerReflection
grpc.reflection.v1alpha.ServerReflection
```

### 发送请求进行测试

```shell
# 参照github.com/envoyproxy/go-control-plane/envoy/service/auth/v3.CheckRequest去构造请求参数
echo '
{
  "attributes": {
    "request": {
      "http": {
        "host": "xxx",
        "path": "xxx",
        "method": "GET",
        "headers": {
          "authorization": "Bearer xxx"
        }
      }
    }
  }
}
' | grpcurl -v -plaintext -d @ localhost:9999 envoy.service.auth.v3.Authorization/Check
```

## 遇到的问题及解决方案

问题是当我返回DenyResponse的时候，同时返回了error，结果istio envoy返回的status code固定是403。通过grpcurl排查发现，如果带了error，则没有DenyResponse的返回。因此需要在接口实现中返回nil。

```go
func (s *extGrpcServer) Check(ctx context.Context, request *authv3.CheckRequest) (*authv3.CheckResponse, error) {
    // ...
    response := &authv3.CheckResponse{
        Status: &status.Status{
            Code: httpStatusCodeToGrpcCode(typev3.StatusCode_Unauthorized),
        },
        HttpResponse: &authv3.CheckResponse_DeniedResponse{
            DeniedResponse: &authv3.DeniedHttpResponse{
                Status: &typev3.HttpStatus{Code: typev3.StatusCode_Unauthorized},
                Headers: []*corev3.HeaderValueOption{
                    {
                        Header: &corev3.HeaderValue{
                            Key:   "Content-Type",
                            Value: "application/json; charset=utf-8",
                        },
                    },
                },
                Body: `{"code":401,"message":"Unauthorized"}`,
            },
        },
    }
    // 这边不能返回error 否则http code会固定是403
    // 所有的deny原因服务都是可控的
    // error message可以在response body中体现
    return response, nil
}

func httpStatusCodeToGrpcCode(code typev3.StatusCode) int32 {
    switch code {
    case typev3.StatusCode_OK:
        return int32(codes.OK)
    case typev3.StatusCode_Unauthorized:
        return int32(codes.Unauthenticated)
    case typev3.StatusCode_Forbidden:
        return int32(codes.PermissionDenied)
    default:
        return int32(codes.Unknown)
    }
}
```
