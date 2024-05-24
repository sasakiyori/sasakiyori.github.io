---
layout: article
title: postgresql数据脱敏
tags: postgresql data anonymous anonymizer rust pgrx
key: 2024-05-24-postgresql-data-anonymous
---

## 背景

数据脱敏是一个很常见的需求，例如对手机号、住址等敏感信息，在普通查询时应该做模糊、改写等操作，避免被不当利用。  
静态数据脱敏是存入数据库的数据直接进行了脱敏处理，是不可逆的，正常只有记录密码之类的才会这么做。  
动态数据脱敏有几种方案，一种是在查询出结果后根据字段对结果集进行更改，另一种就是从源头更改SQL使查询出来的结果就是自带脱敏特性的。  
而从源头更改又有两种方法，一种是直接对文本进行匹配和修改，也就是在客户端侧；一种是在数据库运行时通过hook改变执行语句和执行计划，也就是在服务端侧。  
这次使用的[postgresql anonymizer](https://gitlab.com/dalibo/postgresql_anonymizer)运用到了上述的若干方法，通过集成为数据库插件，方便用户使用类SQL的方式进行配置。  

## 安装

安装参考文档：<https://postgresql-anonymizer.readthedocs.io/en/latest/INSTALL/#install-from-source>  
首先需要前置有rust的环境、安装postgresql所需的一系列库的依赖。然后先下载pgrx(专门用于写postgresql插件的rust库，anonymizer基于这个实现的)  

```shell
# anonymizer要求版本号统一
cargo install cargo-pgrx --version 0.12.0-alpha.1
# 如果有gp的环境变量要先disable掉，否则可能会遇到动态库符号不匹配的情况
# 这个命令会下载pg11~pg16的版本到$HOME/.pgrx目录下，并各自完成initdb
cargo pgrx init
```

然后下载anonymizer库，修改Cargo.toml中默认的数据库为pg12  

```shell
git clone https://gitlab.com/dalibo/postgresql_anonymizer.git
cd postgresql_anonymizer
# 修改Cargo.toml
vim Cargo.toml
```

```toml
# Cargo.toml
# anonymizer仅支持pg12~pg16，默认是pg13
# 这里修改默认为pg12
[features]
default = ["pg12"]
```

然后编译出对应的extension so，安装到pgrx的pg12下的相应目录，就完成了  

```shell
make
# 这里是默认装到$HOME/.pgrx下了。如果要放到自己的目录下，可以考虑修改环境变量或者手动拷贝：涉及control文件、sql文件和anon.so
# 如果有报错，检查一下是否.../lib/postgresql/anon.so的路径是否有错，有可能变成了.../lib/anon.so。可以手动copy一下so文件
make install
# 启动$HOME/.pgrx下的pg12实例
make start
```

连接方式：  
${HOME}/.pgrx/12.19/pgrx-install/bin/psql -d postgres -U postgres -p 28812 -h ${HOME}/.pgrx

## 使用方式

postgresql anonymizer使用了pg12新特性[SECURITY LABEL](https://www.postgresql.org/docs/current/sql-security-label.html)，对表中的column做特殊标签。同时需要我们对特定用户做标签，表明这个用户只能看到脱敏后的数据。对于没有做标签的用户，即使column加了标签也是没有效果的。  

```sql
# 执行此命令后退出session重连
# 或者可以直接配置全局的shared_preload_libraries
ALTER DATABASE test SET session_preload_libraries = 'anon';
# 在db中使用此插件
CREATE EXTENSION IF NOT EXISTS anon CASCADE;
# 初始化插件，主要是将一些用于数据脱敏的假数据存入schema名为anon的一些表中
SELECT anon.init();
# 启动脱敏
SELECT anon.start_dynamic_masking();

# 测试用的表和用户
CREATE TABLE t1(id int, name text, age int);
insert into t1 values (1, 'abcdefg', 2);
CREATE USER testuser password '1';

# 给用户加脱敏标识
SECURITY LABEL FOR anon ON ROLE testuser IS 'MASKED';

# 给表的特定字段设置脱敏，可以用anon已经初始化好的一些脱敏函数
SECURITY LABEL FOR anon ON COLUMN t1.name IS 'MASKED WITH FUNCTION anon.partial(name,2,$$**$$,2)';

# 使用普通用户查询
test=# select * from t1;
 id |  name   | age
----+---------+-----
  1 | abcdefg |   2
(1 row)

# 使用脱敏用户testuser查询
test=> select * from t1;
 id |  name  | age
----+--------+-----
  1 | ab**fg |   2
(1 row)
```

## 实现细节

anonymizer的实现思路是替换SQL、创建视图。

### CURD

对于CRUD，anonymizer本质是通过一系列的function来完成所有的事情。当对表中的某个列做了脱敏标记后，它本质是在schema mask下创建了一个对应的view，相当于写好了对应的脱敏查询sql。  
例如标记为：  
`SECURITY LABEL FOR anon ON COLUMN t1.name IS 'MASKED WITH FUNCTION anon.partial(name,2,$$**$$,2)';`  
这时插件帮我们执行了一系列的SQL：  

```sql
# 根据标记在schema mask下生成同名的view
CREATE OR REPLACE VIEW mask.t1 AS SELECT id,CAST(anon.partial(name,2,$$**$$,2) AS text) AS name,age FROM public.t1;
# 将testuser在public下的权限移除
REVOKE ALL ON SCHEMA public FROM testuser;
# 将anon、mask这两个schema的权限提供给testuser
GRANT USAGE ON SCHEMA anon TO testuser;
GRANT SELECT ON ALL TABLES IN SCHEMA anon TO testuser;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA anon TO testuser;
GRANT USAGE ON SCHEMA mask TO testuser;
GRANT SELECT ON ALL TABLES IN SCHEMA mask TO testuser;
# 将testuser的读取路径默认添加上了mask schema，查找表的时候优先搜索mask再搜索public。即select * from t1，被自动变成了select * from mask.t1
ALTER ROLE testuser SET search_path TO mask,public;
```

在这种效果下，testuser查询t1，就会自动变成查询mask.t1，也就是被导向了被重写了SQL的view中。并且如果尝试指定schema即执行select * from public.t1的话，由于在public下的权限已经被收回，所以会提示permission denied  

### 使用VIEW存在的问题

- 用户一旦被赋予了脱敏标签，就会丧失对schema的操作权限，这个通常不是用户所希望的
- 创建的表只能存在于一个schema中，默认是public。当创建其他schema下的表时，脱敏无法生效，甚至不会生成view。可以通过命令切换默认的schema，再进行生成。流程相对比较繁琐。如果两个schema中有同名的表，就无法使用上述的切换schema的方法，因为会遇到在schema mask下的命名冲突问题。

```sql
# 切换完schema后，退出session重连
ALTER DATABASE test SET anon.sourceschema TO 's1';
# 重连后，执行dynamic_masking命令
SELECT anon.start_dynamic_masking();

# 用testuser连接，可以看到对应的表都是mask schema下的了
test=> \d
              List of relations
 Schema |     Name      |   Type   |  Owner
--------+---------------+----------+----------
 mask   | t1            | view     | pdbadmin
 mask   | t2            | view     | pdbadmin
(2 rows)

test=# \d+ t1
                               View "mask.t1"
 Column |  Type   | Collation | Nullable | Default | Storage  | Description
--------+---------+-----------+----------+---------+----------+-------------
 id     | integer |           |          |         | plain    |
 name   | text    |           |          |         | extended |
 age    | integer |           |          |         | plain    |
View definition:
 SELECT t1.id,
    anon.partial(t1.name, 2, '**'::text, 2) AS name,
    t1.age
   FROM public.t1;

test=# \d+ t2
                               View "mask.t2"
 Column |  Type   | Collation | Nullable | Default | Storage  | Description
--------+---------+-----------+----------+---------+----------+-------------
 id     | integer |           |          |         | plain    |
 name   | text    |           |          |         | extended |
 age    | integer |           |          |         | plain    |
View definition:
 SELECT t2.id,
    anon.partial(t2.name, 2, '**'::text, 2) AS name,
    t2.age
   FROM s1.t2;

# 查询到的也都是脱敏的数据
test=> select * from t1;
 id |  name  | age
----+--------+-----
  1 | ab**fg |   2
(1 row)

test=> select * from t2;
 id |  name  | age
----+--------+-----
  1 | aa**aa |   1
(1 row)
```

### COPY

对于transcation state的copy操作(例如copy t1 to stdout)，通过pg的ProcessUtility_hook来做utility stmt的实时替换。  

```c
typedef void (*ProcessUtility_hook_type) (PlannedStmt *pstmt,
                                          const char *queryString,
                                          bool readOnlyTree,
                                          ProcessUtilityContext context,
                                          ParamListInfo params,
                                          QueryEnvironment *queryEnv,
                                          DestReceiver *dest,
                                          QueryCompletion *qc);
```

具体使用方式为：  

```sql
# 使用时需要打开GUC：
set anon.transparent_dynamic_masking=true;

# 例如执行上述已经被设置脱敏标签的t1：
# 脱敏目前只支持 COPY {TABLE} TO 的格式
# COPY FROM和COPY (SELECT ...) TO都不支持
COPY t1 TO stdout;

# 动态替换实际上就是把mask.t1展开了(因为view不能直接用作copy)：
SELECT id AS id, name AS name, age AS age FROM mask.t1;
```

anonymizer的日志为：  

```shell
DEBUG:  Anon: COPY found
# 替换前的copy stmt
DEBUG:  Anon: copystmt before = CopyStmt { type_: T_CopyStmt, relation: 0x5585b177e660, query: 0x0, attlist: 0x0, is_from: false, is_program: false, filename: 0x0, options: 0x0, whereClause: 0x0 }
# 通过relation构造query
DEBUG:  Anon: Query = SELECT id AS id, name AS name, age AS age FROM mask.t1;
# 通过query string构造copy raw stmt
DEBUG:  Anon: Copy raw_stmt = RawStmt { type_: T_RawStmt, stmt: 0x5585b1840410, stmt_location: 0, stmt_len: 54 }
# 替换后的copy stmt。可以看到这里从relation切换成了query
DEBUG:  Anon: copystmt after = CopyStmt { type_: T_CopyStmt, relation: 0x0, query: 0x5585b1840410, attlist: 0x0, is_from: false, is_program: false, filename: 0x0, options: 0x0, whereClause: 0x0 }
```
