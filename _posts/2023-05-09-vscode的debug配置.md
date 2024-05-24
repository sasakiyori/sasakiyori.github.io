---
layout: article
title: vscode的debug配置
tags: vscode debug gdb lldb
key: 2023-05-09-vscode-debug-settings
---

## 背景

有些时候我们想要直接在vscode上实现开发+调试+运行。

## 开发配置

对于部分语言(例如c)，我们需要指定一些配置让vscode识别到它的代码、静态库、动态库等依赖，才能正常使用。不然会有很多的报错，也无法正常进行代码跳转。  
我们可以通过配置`c_cpp_properties.json`和`settings.json`来解决。  
这边以mac环境下的[pgbouncer](https://github.com/pgbouncer/pgbouncer)代码为例，配置两个文件：

```json
// c_cpp_properties.json
{
    "configurations": [
        {
            "name": "Mac",
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/local/include/**",
                "/usr/local/opt/openssl/include/**",
            ],
            "defines": [
                "USUAL_LIBSSL_FOR_TLS",
                "USE_TLS"
            ],
            "macFrameworkPath": [
                "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks"
            ],
            "compilerPath": "/usr/bin/clang",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "macos-clang-x64",
            "configurationProvider": "ms-vscode.makefile-tools"
        }
    ],
    "version": 4
}
```

```json
// settings.json
{
    "files.associations": {
        "bouncer.h": "c",
        "system.h": "c",
        "base.h": "c",
        "concepts": "c",
        "functional": "c",
        "iterator": "c",
        "locale": "c",
        "string": "c",
        "base_win32.h": "c",
        "mbuf.h": "c",
        "memory_resource": "c",
        "tls_internal.h": "c",
        "scram.h": "c",
        "ctype.h": "c",
        "config_msvc.h": "c",
        "resource.h": "c",
        "_uint32_t.h": "c",
        "chrono": "c",
        "unordered_map": "c",
        "ranges": "c",
        "list.h": "c",
        "event_struct.h": "c",
        "logging.h": "c",
        "string.h": "c",
        "tls_compat.h": "c",
        "random": "c",
        "__node_handle": "c",
        "fileutil.h": "c",
        "typeinfo": "c",
        "socket.h": "c",
        "pam.h": "c",
        "__bit_reference": "c",
        "atomic": "c",
        "bitset": "c",
        "cstddef": "c",
        "deque": "c",
        "__memory": "c",
        "limits": "c",
        "optional": "c",
        "ratio": "c",
        "system_error": "c",
        "tuple": "c",
        "type_traits": "c",
        "vector": "c",
        "sstream": "c",
        "signal.h": "c",
        "objects.h": "c",
        "proto.h": "c",
        "__string": "c",
        "dnslookup.h": "c",
        "loader.h": "c",
        "takeover.h": "c",
        "iobuf.h": "c",
        "util.h": "c",
        "__config": "c",
        "pthread.h": "c",
        "__locale": "c",
        "__threading_support": "c",
        "string_view": "c",
        "variant": "c",
        "array": "c",
        "span": "c"
    }
}
```

## debug配置

在一个项目的根目录创建文件夹`.vscode`，并在底下创建`launch.json`文件。

### 二进制模式

比较通用的二进制模式。指定二进制目录和代码目录即可debug。

```json
// launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug App",                                // debug程序的名称
            "type": "lldb",                                     // debug使用的工具 比如lldb, lldb-mi, gdb等
            "request": "launch",                                // 要工具做的事 可选项是launch和attach launch比较方便
            "program": "${workspaceFolder}/AppBin",             // 二进制文件路径
            "args": ["-arg", "test", "-v"],                     // 启动二进制文件时需要的参数
            "cwd": "${workspaceFolder}",                        // 工作路径
        }
    ]
}
```

### 代码模式

对于某些语言(比如golang)，可以直接指定启动项进行启动，不需要进行编译，即不需要指定二进制文件。

```json
// launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug App",                                // debug程序的名称
            "type": "go",                                       // debug使用的工具 比如lldb, lldb-mi, gdb等
            "request": "launch",                                // 要工具做的事 可选项是launch和attach launch比较方便
            "program": "${workspaceFolder}/main.go",            // golang入口文件
        }
    ]
}
```

## 运行前自动进行编译

对于二进制运行的模式，因为是分别制定二进制文件和代码目录的方式，在开发时容易忘记重新编译二进制文件导致代码指向有问题。  
因此我们需要运行时自动重新编译的命令，可以用`tasks.json`配合`launch.json`的配置做这件事。  

```json
// tasks.json
{
    "tasks": [
        {
            "type": "shell",                    // 指定在shell做这件事
            "label": "build",                   // 指定标签名 launch.json的配置需要使用它
            "command": "make",                  // 你需要做的任务的命令
            "args": [                           // 你需要做的任务的入参
                "-C",
                "${workspaceFolder}",
                "-j16"
            ],
        }
    ],
    "version": "2.0.0"
}
```

```json
// launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug App By lldb",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/AppBin",
            "args": ["-arg", "test", "-v"],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "build"                    // 在执行launch之前所需要做的任务label
        },
        {
            "name": "Debug App By gdb",
            "type": "cppdbg",
            "request": "launch",
            "MIMode": "gdb",
            "program": "${workspaceFolder}/AppBin",
            "args": ["-arg", "test", "-v"],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "build",                   // 在执行launch之前所需要做的任务label
        }
    ]
}
```

## 多服务配置

主要依赖vscode workspace实现：<https://code.visualstudio.com/docs/editor/multi-root-workspaces>  
例如对于多个golang的repo，能做到检测多个`go.mod`，且不会互相冲突，可以同时启动多个服务  

### workspace创建

- 方法1：在vscode的`资源管理器界面`右键，选择`将文件夹添加到工作区`，选择你想要打开的module即可重复上述过程即可添加多个module到工作区，此时vscode的工作区自动转换为multi-root模式  
- 方法2：直接创建一个`.code-workspace`后缀的文件，配置文件中的`folders`标签就配置了所有你添加到工作区的module，其中每个文件夹的位置由`path`标签来定义，它是一个相对地址  

### 示例

假设当前的repo分布为：  

```shell
.
|-- my.code-workspace
|-- server1
|-- server2
|-- server3
```

那么对`my.code-workspace`可以进行如下配置：  

```json
{
    // repo 与 my.code-workspace 的相对路径
    "folders": [
        {"path": "server1"},
        {"path": "server2"},
        {"path": "server3"},
    ],
    // 一些插件和测试用环境变量的配置
    "settings": {
        "liveServer.settings.multiRootWorkspaceName": "pkg",
        "go.testEnvVars": {
            "MY_TEST_ENV": "abc"
        },
    },
    "launch": {
        "version": "0.2.0",
        // 可以对多个服务进行组合
        "compounds": [
            {
                "name": "组合1",
                "configurations": ["server1", "server2"],
                // 关闭时是否全关
                "stopAll": true,
            },
            {
                "name": "组合2",
                "configurations": ["server1", "server3"],
                "stopAll": false,
            }
        ],
        // 每个服务自己的配置
        "configurations": [
            {
                "name": "server1",
                "type": "go",
                "request": "launch",
                "mode": "auto",
                // 注意这里的启动方式
                "program": "${workspaceRoot:server1}/main.go",
                // 服务自己的启动用环境变量
                "env": {
                    "MY_SERVER_ENV": "aaa"
                }
            },
            {
                "name": "server2",
                "type": "go",
                "request": "launch",
                "mode": "auto",
                "program": "${workspaceRoot:server2}/main.go",
                "env": {
                    "MY_SERVER_ENV": "bbb"
                }
            },
            {
                "name": "server3",
                "type": "go",
                "request": "launch",
                "mode": "auto",
                "program": "${workspaceRoot:server3}/main.go",
                "env": {
                    "MY_SERVER_ENV": "ccc"
                }
            }
        ]
    }
}
```
