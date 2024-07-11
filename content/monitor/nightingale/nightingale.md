+++
title = 'Nightingale'
date = 2024-07-04T08:56:28+08:00
draft = true
+++

夜莺：侧重告警

+ Grafana 侧重看图，夜莺侧重告警
+ 可以接入多种不同的数据源，比如 Prometheus、VictoriaMetrics、M3DB、ElasticSearch、Loki、TDEngine 等等，在夜莺中配置管理告警规则，夜莺周期性去查询各个存储，判定异常数据，产生告警事件，然后把告警事件通过钉钉、企微、邮件等方式发出

很多公司是一套组合方案（成年人的世界，没有非黑即白，都要）：

- 数据采集：组合使用了各种 agent 和 exporter，比如使用 [Categraf](https://github.com/flashcatcloud/categraf)，辅以各类 Exporter
- 存储：时序库主要使用 VictoriaMetrics，因为 VictoriaMetrics 兼容 Prometheus，而且性能更好且有集群版本，对大部分公司，单机版就足够用了
- 告警引擎：使用夜莺，方便不同的团队管理协作，内置了一些规则开箱即用，告警规则的配置比较灵活
- 看图可视化：使用 Grafana，图表更为炫酷，社区非常庞大，从 Grafana 站点可以找到很多别人做好的仪表盘，直接导入即可
- 告警事件 OnCall 分发：使用 [FlashDuty](https://flashcat.cloud/product/flashduty/)，聚合了 Zabbix、Prometheus、夜莺、Open-Falcon、云监控、Elastalert 等各类告警事件，统一聚合降噪、排班、认领升级等



完成了运维二梯队新兵训练营的学习，熟悉了培训体系的内容和操作流程，并按照培训手册中的要求完成了相关作业

百胜旗下品牌，IT团队介绍，消费者应用运维定级、生产环境介绍，百胜运维平台，运维“高压线“





