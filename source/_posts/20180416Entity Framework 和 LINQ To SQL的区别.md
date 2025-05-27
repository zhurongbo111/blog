---
title: Entity Framework 和 LINQ To SQL的区别
date: 2018-04-16 16:10:00
categories:
 - 服务端
tags:
 - C#/.Net
 - Entity Framework
description: 比较了Entity Framework 和 LINQ To SQL各方面的优劣势
---
# Entity Framework 和 LINQ To SQL的区别

## 综述

- **LINQ**是一种语言集成查询，它包含了：
  - LINQ to SQL
  - LINQ to Objects
  - LINQ to XML
  - LINQ to Entities（Entity Framework两种查询方式之一，另外一种叫做Entity SQL）
  
- **Entity Framework** 是一种ORM （Object Relational Mapping）框架，把关系型数据转换成对象的一种框架。
  
- **LINQ to SQL**是Linq最初提供的一种访问数据的方式，它允许你从SQL Server数据库获取数据。
  
- **LINQ to Entities**是Linq提供的另外一种访问数据库的方式。和LINQ to SQL不同的是，它支持的数据库类型和ADO.Net支持的数据库一样多。

### 什么时候使用Entity Framework 和 LINQ to SQL

当你的应用程序符合以下条件时，就可以使用Entity Framework:
- 需要比较灵活的映射关系
- 能够查询除SQL Server系列产品以外的关系型数据库
- 能够在SSRS，BI，SSIS之间可复用的数据访问模式
- 需要提供完整的文本查询语言
- 需要查询概念模型的能力

如果你不需要以上的各种功能，那么LINQ to SQL是一种比较简单的快速开发选择。

### 两种方案的功能比较

| 功能 | LINQ to SQL | Entity Framework |
| --- | --- | --- |
| 模型（Model） | 基于数据库实体 | 基于概念模型 |
| 数据库 | 只支持SQL Server | 大部分的数据库 |
| 复杂度 | 简单 | 复杂 |
| 开发周期 | 快速开发 | 开发缓慢，但是功能更强大 |
| 查询方式 | LINQ to SQL (for select), Data Context | LINQ to Entities (for select), Entity SQL, Object Services (for update, create, delete, store procedure, view), Entity Client (is an ADO.NET managed provider, it is similar to SQLClient, OracleClient) |
| 当数据库发生改变 | 不支持同步 | 支持同步 |
| 发展性 | 微软不再开发新功能 | 微软主推的ORM框架 |
| 自动生成数据库 | 不支持 | 支持 |
| 性能 | 第一次较慢 | 第一次较慢，整体会优于前者 |

### 性能比较

相对于Entity Framework，LINQ to SQL 是更轻量级的框架，EF需要处理两层模型映射，而LINQ to SQL只有一层映射。EF会生成更多更复杂的TSQL，这些TSQL都是作用于更好的可读性，同时SQL在大多数情况会得到相同的执行计划。但是部分情况下会生成更大更复杂的SQL语句，因此会有性能影响。

而LINQ to SQL在客户端有轻微的查询优化，它会评估where子句进行优化，所以会有更好的查询效率。

### LINQ to SQL是否已经被丢弃？

虽然网上有很多人讨论到此功能已经被弃用，但是该功能还是存在于.Net Framework之中，不过该功能完全可以被Entity Framework取代，并且具有更少的限制。

EF是主流，也是微软主推的框架，不过LINQ to SQL也还是被微软支持的。根据不同的情况，可以进行不同的选择。

原文地址：[https://maxivak.com/entity-framework-vs-linq-to-sql/](https://maxivak.com/entity-framework-vs-linq-to-sql/)
