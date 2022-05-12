---
title: "Federation"
description : "Prometheus 简单的 Federate 配置"
isCJKLanguage: true
date: 2022-05-12T10:23:23+08:00
#lastmod: 2022-04-28T15:13:38+08:00
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**

# 参考
[operater additional scrape config](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/additional-scrape-config.md)  
[federation](https://prometheus.io/docs/prometheus/latest/federation/)

# 实操

## 附加的配置文件

>prometheus-additional.yaml
```yaml
- job_name: 'federate'
  scrape_interval: 15s

  honor_labels: true
  #metrics_path: '/federate'
  # 因环境通过反代出来的，path 是这个
  # ip/prometheus/federate
  metrics_path: '/prometheus/federate'

  params:
    'match[]':
    #- '{job="prometheus"}'
    #- '{__name__=~"job:.*"}' 
    #- '{job!=""}'
    - '{job=~"apisix"}'
    - '{job=~"kube-state-metrics"}'
    - '{job=~"node-exporter"}'
    - '{job=~"rook-ceph-mgr"}'
    - '{job=~"kubelet"}'

  static_configs:
    - targets: ['172.30.3.229']
      # 添加额外的 label
      #labels:
      #  k8scluster: 172.30.3.229
    - targets: ['172.30.3.230']
      #labels:
      #  k8scluster: 172.30.3.230
    - targets: ['172.30.3.222']
      #labels:
      #  k8scluster: 172.30.3.222

```

## 创建附加的配置

```shell
kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml --dry-run -oyaml > additional-scrape-configs.yaml
kubectl apply -f additional-scrape-configs.yaml -n monitoring

```

## 配置
```shell
kubectl -n monitoring get  prometheus 
NAME   VERSION   REPLICAS   AGE
k8s    2.28.0    2          40h
hchen@node1:~$ kubectl -n monitoring edit  prometheus  k8s
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
..............................................
spec:
..........................................
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
....................................................
```

# curl 测试

```shell
curl ip:9090/federate?match[]={job!=""}

```