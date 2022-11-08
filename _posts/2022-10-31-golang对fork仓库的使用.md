---
layout: article
title: golang对fork仓库的使用
tags: golang fork repository package
key: 2022-10-31-the-usage-of-repository-forked-from-github
---

## 背景
总会有一些golang package在实际场景下不满足需求，且改动点有可能是针对自身业务场景特定的，无法通过在github提issue或者pr的方式解决问题。
这个时候可能就需要fork对应的仓库，并进行独立维护。(当然要注意对应repo的license是否允许你去这样做)

## 维护
#### fork仓库
   这边按已有的[sasakiyori/pgx](https://github.com/sasakiyori/pgx)为例，fork自[jackc/pgx](https://github.com/jackc/pgx)。
   可以设置脚本定时拉取最新代码，例如github可以设置action

#### 开发
   在fork的仓库中，按分支开发。
   go的模块管理依赖于git tag，因此分支开发时，要先确定自己要基于什么版本开发，可从`go.mod`中查看。
   例如，`pgx`基于`v4.16.1`版本进行额外开发:
   ```shell
   git fetch --all --tags --prune
   git checkout -b v4-test tags/v4.16.1
   ```

#### 发布
   当分支上的修改测试完成后，需要打上标签，才能让调用方识别到：
   ```shell
   # git tag [tag_name] [branch_name]
   git tag v4.16.1-patch1-test v4-test
   git push origin v4.16.1-patch1-test
   ```
   因为在后续提到的`package replace`过程中，版本号是有格式的强制要求的，所以可以考虑tag的命名方式为 原repo的tag+序列号+修改主要信息。
   (默认规范是tag+时间+commit SHA， 但是改成上述格式可能较为直观)
   ```shell
   replace: must be of the form v1.2.3
   ```

## 使用
1. 如果fork的repo是私有的(github默认为public，一些其他平台可能默认为private)，则需要在环境变量中增加对应路径或前缀：
   ```shell
   # 或者: export GOPRIVATE=github.com/sasakiyori
   # 或者: go env -w GOPRIVATE=github.com/sasakiyori/pgx
   export GOPRIVATE=github.com/sasakiyori/pgx

   # 必须为package的格式，比如下面这个按路径的写法加/，是不会生效的
   # export GOPRIVATE=github.com/sasakiyori/
   ```

2. 在go.mod中将对应的package进行替换。具体格式含义可参考：<https://go.dev/ref/mod#go-mod-file-replace>。要注意由于tag是v4开头，因此package中也需要加上v4
   ```go
   require (
       github.com/jackc/pgx/v4 v4.16.1
   )

   replace (
       github.com/jackc/pgx/v4 => github.com/sasakiyori/pgx/v4 v4.16.1-patch1-test
   )
   ```

## 参考
   - <https://stackoverflow.com/questions/14323872/using-forked-package-import-in-go>
   - <https://stackoverflow.com/questions/27500861/whats-the-proper-way-to-go-get-a-private-repository>
