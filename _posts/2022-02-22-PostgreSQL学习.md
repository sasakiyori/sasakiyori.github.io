---
layout: article
title: PostgreSQL学习
tags: PostgreSQL psql postgres pgx
key: 2022-02-22-PostgreSQL
---

## 前言
持续更新中


## 基础使用
### 参考文档
1. 官方手册:
   - [英文版](https://www.postgresql.org/docs/current/)
   - [中文版](http://www.postgres.cn/docs/9.3/index.html)



## [pq](https://github.com/lib/pq)
几年前比较流行的纯go语言编写的工具, 现维护不频繁, 官方readme建议迁移到pgx的使用:
> For users that require new features or reliable resolution of reported bugs, we recommend using pgx which is under active development.


## [pgx](https://github.com/jackc/pgx)
纯go语言编写的PostgreSQL工具, 目前较活跃, 功能较齐全

### 官方文档
https://pkg.go.dev/github.com/jackc/pgx/v4

### 具体应用
1. 使用copyFrom直接导入csv
   - 参考: https://stackoverflow.com/questions/66779332/bulk-insert-from-csv-in-postgres-using-golang-without-using-for-loop
   - demo:
        ```go
        f, err := os.Open(filename)
        defer(f.Close())
        // 如果csv文件第一行是表对应的字段名的话 需要添加HEADER选项用来跳过第一行的读取
        sql := fmt.Sprintf("COPY %s FROM STDIN (FORMAT csv, HEADER)", tablename)
        res, err := dbconn.Conn().PgConn().CopyFrom(context.Background(), f, sql)


        // 如果导入的csv的列有缺失(即非全表导入) 则需要指定列名
        f, err := os.Open(filename)
        defer(f.Close())
        scanner := bufio.NewScanner(transform.NewReader(csv, unicode.UTF8BOM.NewDecoder()))
	    scanner.Scan()
        columnnames := scanner.Text()
        sql := fmt.Sprintf("COPY %s (%s) FROM STDIN (FORMAT csv, HEADER)", tablename, columnnames)
        // offset重置到文件头 进行copy操作
        csv.Seek(0, io.SeekStart)
        res, err := dbconn.Conn().PgConn().CopyFrom(context.Background(), f, sql)
        ```