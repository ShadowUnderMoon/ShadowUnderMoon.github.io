---
title: "Prometheus_and_grafana"
author: "爱吃芒果"
description: 
date: 2025-02-01T19:42:56+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:	
  - prometheus
  - grafana
---

## Prometheus

https://www.youtube.com/watch?v=h4Sl21AKiDg

![image-20250201195125453](prometheus_arch.png)

![image-20250201195720822](prometheus_metrics.png)

![image-20250201195956124](prometheus_collect.png)

![image-20250201200134497](prometheus_exporter.png)

![image-20250201201116934](prometheus_config.png)

![image-20250201201412269](prometheus_alert.png)

Prometheus Server, Pushgateway, Alertmanager

https://prometheus.io/docs/concepts/metric_types/

https://itnext.io/prometheus-for-beginners-5f20c2e89b6c



**Prometheus is essentially just another metrics collection and analysis tool,** and at its core it is made up of 3 components:

- A [**time series database**](https://en.wikipedia.org/wiki/Time_series) that will store all our metrics data
- A **data retrieval worker** that is responsible for **pulling/scraping** metrics from external sources and **pushing** them into the database
- A web server that provides a **simple web interface** for configuration and querying of the data stored.

![image-20250201214518713](prometheus_arch_all.png)

https://prometheus.io/docs/practices/naming/#metric-names

https://prometheus.io/docs/practices/naming/#base-units