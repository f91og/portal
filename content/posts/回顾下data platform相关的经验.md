+++
title = ' 我的 Data Platform 工程经验总结 '
date = 2025-12-11T09:58:08Z
tags = [' 架构 ']
categories = ['data', 'Infrastructure']
draft = false
+++

目前工作经验中 data platform 占据了很大一部分，也是时候来回顾工作总结下这些经验了。

<!--more-->
## M 公司

工业商品 b2b 业务，数据量没那么大，用 GCP 的下一些工具外加 python，go 完全足够。

和 data platform 相关的有 2 个，一个是在线特征平台，还有一个是日志收集系统，主要技术栈就是 GO，Python 和 GCP.

关于在线特征平台：在线特征平台就是把模型或业务需要的“关键数据”整理好，放进一个可以低延迟（<10ms）查询的数据库里，让线上系统（推荐/风控/广告/个性化）能在几毫秒内拿到特征数据（最近 30 天点击次数， 商品热度，用户平均浏览价格，风控需要的最近 N 次付款行为等）。

公司的在线特征平台的基本设计就是将 bigquery 的数据导入到 bigtable 里并用 go 来开发了一个特征数据的获得 api，这样推荐系统可以直接通过这个 api 来拿到特征数据，一些要点：
- 整个系统由 Go（Serving）、Python（Orchestration）、Java Beam（Dataflow）和 GCP 服务协同构成
- 数据流 BigQuery → Dataflow Job → Bigtable
- 事件驱动来 ETL 调度，Flask API 会在收到某些请求（或按 schedule）时 **publish 一个 Pub/Sub message**, 后台的 worker/subscriber 会订阅特定 topic 来触发 dataflow job，这样不需要通过 cronjob 或者手动来，更加灵活
- 生产与测试环境的流量回放，Pub/Sub 用来存放线上真实流量（请求参数），然后用于流量回放到 staging 或 load test 环境
- Go API 提供 Serving 层，根据 YAML 注册路由 (动态生成 api endpoints),点查 Bigtable(可选 Redis 缓存), 返回给业务端（推荐系统等）
简单说就是从离线特征存储 → 在线特征存储，并提供一个访问层，

为什么要多此一举搞一个 Go 的 Serving 层？为什么不让其他系统（推荐等）直接调用 Bigtable API？
- 避免其他团队重复造轮子：row key 规则，反序列化/解析，缓存，数据清洗/默认值处理，重试/超时/backoff，查询逻辑（Bigtable 这种无 schema 的列式存储重查询逻辑需要业务方自己拼）等。
- 避免把 Bigtable 的内部 schema 暴露给外部系统，Bigtable row key 规则一旦变动或特征字段重命名、增加、删除，那么所有依赖方（推荐系统、实验平台、AB 测试）都要改代码、重新上线、重新验证，紧耦合
- Go serving 层就是一个 Schema + API Contract 屏障，无论 Bigtable schema 如何演进，对外可保持 API 不变， `GET /feature/user?uid=123`
- 提供“统一特征计算逻辑”，防止 N 个系统重复处理 N 遍，默认值填充，类型转换（int → float）等
- 统一性能优化层：缓存、批量查询、熔断、限流
- 多系统访问 Bigtable 时的权限、安全问题

关于特征存储的解释：[Feature Storeについてふんわり理解する - Re:ゼロから始めるML生活](https://www.nogawanogawa.com/entry/feature_store)

日志收集数据就是将用户行为日志给扔到 bigquery 里
- 用户的 Web/App 行为（page view、click、performance timing 等）会发送到 Flask API, 作下校验 (payload 大小与 header（IP、UA 等), 直接丢到 Pub/Sub（tracking-log topic）
- 长驻进程订阅 `tracking-log` 的 raw events, 进行字段解析，丢弃非法/空字段, schema 标准化等，将处理后的数据发布给 Pub/Sub（parsed-logs topic）
- 使用 Apache Beam Streaming（Dataflow runner）读取 `parsed-logs`，做实时聚合，如性能指标（如 95/99 百分位加载时间），页面级 PV/UV 统计，聚合后的结果以 JSON 形式写入 Pub/Sub log-statistics topic
- `log-statistics` 事件触发 Cloud Function, 实时监控输出到 datadog
- parsed-logs 会定期写到 BigQuery 分区表 (历史存储)，支持 BI/分析师做历史行为分析、漏斗分析，也可以用于模型训练（用户画像、推荐系统行为特征）
## P 公司

扫码支付公司，数据量大，需要用的一些大数据的技术栈。

data platform 的 infra 基本就是 cdc tool -> kafka -> spark ->hive metastore，presto， hudi (数据湖) -> (BQ, ES 等)，job 调度工具是 airflow。
- infra 用的是 AWS EMR，自带大数据的一些工具，如 Hadoop，Spark，hive，presto，hudi 什么的
- apache canal 作为 cdc tool 来监听 binlog，发给 Kafka，Spark 消费，给 hudi，比较标准的做法
- Spark + Hudi 是写入层，存储底层是 S3，不光是数据库中的数据，一些文件比如 csv 也会被聚合到 Hudi 里
- Presto 是数据湖查询引擎，用 SQL 查询存放在 S3 上的 Hudi 表（或 Parquet 数据），无需将数据加载到数据库中，也无需提前复制数据，不需要启动 job 或者 ETL 的查询层，速度极快
- Hive Metastore 识别表结构 / 分区信息，Presto 本身不存 schema，它依赖 Hive Metastore，Hive Metastore 告诉 Presto 表有哪些字段、哪些分区、文件路径是什么
- airflow 作为 job 调度工具，上游 team 需要作每日报表这样的任务时通过 airflow 触发 job 来用 hudi 中的数据来生成报表
- 后来 leader 决定迁移到 AWS GLUE + lambda，觉得 serverless 会减少 platform 运维成本，提提高自动化，后来证明 leader 想多了，大数据量的场景下明显 serverless 不太适合，光 spark 版本问题就等 AWS 的人来解决好久，更别说 serverless 自带可能的延迟，debug 困难和参数调优受限了

Hudi 的作用是在对象存储（S3）上实现“可更新的数据湖”（包括 upsert、删除、增量读、事务一致性、表时间线管理），没有 Hudi，数据湖只能 append，不能可靠地更新
- 数据从 Canal/Kafka 来，是每天/每小时/实时都在发生变更的
- 对已有行做更新/覆盖，即 upsert
- 支持精确一次语义（exactly-once）
- 供增量查询（Incremental Query）
- Hudi 在 S3 上模拟了 **ACID 事务语义**，提供 MOR/COW 两种存储模式，读写性能可调

和 iceberg 比较而言，Hudi 是为“不断更新”的数据设计的；Iceberg 是为“大规模分析查询”设计的，如果数据是“堆积为主、分析为主”，用 Iceberg.

在这个公司没有涉及到特征平台是因为当时他们还有没有和用户画像相关的功能，如推荐等，就是个单纯的扫码支付，所有数据平台主要支撑的是他们的日常报表，bi 等。
## D 公司

创业公司，通过自己的特征平台和非监督算法提供金融反诈业务，infra 是基于 AWS 的自建 k8s 集群，在里面部署 yugabyte，clickhouse，flink, spark, kafka，mysql 等。
- 金融反诈的流程是：在一笔交易发生的几十毫秒内，综合“当前状态 + 最近行为 + 历史模式”，判断这笔交易要不要放行，点查用户状态 -> 获取“最近行为特征”（近实时统计，最近 N 次交易金额分布等）-> 规则判断（Rule Engine）-> 模型评分（ML Inference）-> 决策（Decision）
- 交易事件写入 Kafka，通过 Flink 做实时特征与在线状态更新；Spark 做离线特征、历史回放、模型训练与大规模重算
- Yugabyte 存储用户 / 账户 / 设备的当前状态，反诈规则与配置（白名单黑名单等），用于点查（按 key 查，不是扫描表查），这些数据必须强一致，必须要用有强一致性保证的数据库
- ClickHouse 这种擅长扫描大量行做聚合、统计的用于行为分析 / 特征统计，因为要综合用户过去的行为记录来分析（要扫描大量的历史记录），用来承载高吞吐的支付行为数据和近实时统计特征，支持复杂聚合和模型分析
- 多 k8s 集群中的 mysql 部署用的是 statefulset，然后使用 aws volume 来作为 db 的存储卷，数据迁移的时候倒是挺方便，直接换 pv 就可以了
- 能写入的 mysql 同时只有一个，通过在主副集群里都用主集群的 mysql 的 svc 来做的，读的话可以同时读
- 主副集群中 kafka 的消息同步通过 Kafka MirrorMaker 来实现
- Spark，Flink 啥都都部署在 k8s 里，需要跑 job 就用 k8s 的 job 或者 pod 来新起一个
- job 调度用的是 luigi，和 airflow 相比，Luigi 是轻量稳定的 Python pipeline 框架，更像一个简单可靠的“任务依赖执行器”，更加轻量，就是一个 Python package，而 Airflow = 调度系统（Scheduler） + Workflow UI + Operator Ecosystem，显然不适合 startup 这种人少，job 又不多的场景
- 集群的设计中需要对 outbound 也要有所限制，不能让它随便 call 外面的 api

这个公司也有一个特征平台，用于金融反诈的实时分析用的，大概就是用户最近的支付频率，地点，设备什么的。
## 总结

虽然架构不同，但是要解决的问题一致，就是把分散的数据聚合到一起然后给其余部门使用，让这些数据产生价值，基本要点就是 ETL，一个数据中心（数据仓库 or 数据湖），然后一个 job 调度工具。
