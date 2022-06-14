---
title: "Grafana"
description : "Grafana 随记"
isCJKLanguage: true
date: 2022-05-27T13:53:23+08:00
lastmod: 2022-06-14T19:13:38+08:00
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---
> **Have a nice day! :heart:**

# 迁移配置到其他环境
## 参考

[How to migrate your configuration database](https://grafana.com/blog/2020/01/13/how-to-migrate-your-configuration-database/)

## 前言

> grafana 不会对数据库有太多复杂的操作，也就保存一些基本的配置信息(比如 users, dashboards, alerts 等等)，所以默认选择 sqlite3 作为配置的存储。

## 迁移当前配置

> 在当前环境中一条一条的添加了很多的配置，将来新的环境也可以使用相同的配置，那是否还是需要一条一条的去添加了？ 当然可以，鱼就是这样摸的。但鱼被其他伙伴摸完了，无鱼可摸了，那怎么办？ 那就，那就开始正文吧。

### 当前的数据库情况

> 当前环境部署在 k8s 集群里面, 版本 Grafana v8.2.2 (6232fe07c0)

![](/static/images/k8s/prometheus/grafana-alerts-rules.png)

```shell
# 进入 pod
kubectl -n monitoring exec -it $(kubectl -n monitoring  get pods -l "app.kubernetes.io/component=grafana" -oname | awk -F'/' '{print $NF}') -- bash
# 默认位置在 /var/lib/grafana
# 用户和组都是 nobody, 权限 0660
cd /var/lib/grafana/
ls -l grafana.db
-rw-rw----    1 nobody   nobody     2560000 May 27 07:36 grafana.db
```
### 导出当前数据库
```shell
# 标配的 cp 
kubectl -n monitoring cp $(kubectl -n monitoring  get pods -l "app.kubernetes.io/component=grafana" -oname | awk -F'/' '{print $NF}'):/var/lib/grafana/grafana.db ./grafana.db
tar: removing leading '/' from member names
# 用户、属组、权限都变了
ls -l grafana.db 
-rw-rw-r-- 1 handsome handsome 2560000 May 27 15:40 grafana.db
# 修改用户、权限，要不然跑起来的时候会因为权限问题而不能正常的访问
sudo chown nobody:nobody grafana.db # ......
chown: invalid group: 'nobody:nobody'
sudo chown nobody grafana.db 
sudo chmod 0660 grafana.db
# 组，这没 nobody 的组哒，上面都报错了，算了，认了，不是一个严谨的人
ls -l grafana.db 
-rw-rw---- 1 nobody handsome 2560000 May 27 15:40 grafana.db
file grafana.db 
grafana.db: SQLite 3.x database, last written using SQLite version 3035004

```

### 伪迁移

> 为撒叫伪迁移呢？ 因为准备直接 docker run 一个而不是搭建一套 k8s

```shell
docker run -it --rm -p 3000:3000 --user nobody -e GF_UNIFIED_ALERTING_ENABLED=true -e GF_ALERTING_ENABLED=false -v $PWD/grafana.db:/var/lib/grafana/grafana.db grafana/grafana:8.2.2
WARN[05-27|08:04:24] falling back to legacy setting of 'min_interval_seconds'; please use the configuration option in the `unified_alerting` section if Grafana 8 alerts are enabled. logger=settings
WARN[05-27|08:04:24] falling back to legacy setting of 'min_interval_seconds'; please use the configuration option in the `unified_alerting` section if Grafana 8 alerts are enabled. logger=settings
INFO[05-27|08:04:24] Config loaded from                       logger=settings file=/usr/share/grafana/conf/defaults.ini
INFO[05-27|08:04:24] Config loaded from                       logger=settings file=/etc/grafana/grafana.ini
INFO[05-27|08:04:24] Config overridden from command line      logger=settings arg="default.paths.data=/var/lib/grafana"
INFO[05-27|08:04:24] Config overridden from command line      logger=settings arg="default.paths.logs=/var/log/grafana"
INFO[05-27|08:04:24] Config overridden from command line      logger=settings arg="default.paths.plugins=/var/lib/grafana/plugins"
INFO[05-27|08:04:24] Config overridden from command line      logger=settings arg="default.paths.provisioning=/etc/grafana/provisioning"
INFO[05-27|08:04:24] Config overridden from command line      logger=settings arg="default.log.mode=console"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_PATHS_DATA=/var/lib/grafana"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_PATHS_LOGS=/var/log/grafana"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_PATHS_PLUGINS=/var/lib/grafana/plugins"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_PATHS_PROVISIONING=/etc/grafana/provisioning"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_UNIFIED_ALERTING_ENABLED=true"
INFO[05-27|08:04:24] Config overridden from Environment variable logger=settings var="GF_ALERTING_ENABLED=false"
INFO[05-27|08:04:24] Path Home                                logger=settings path=/usr/share/grafana
INFO[05-27|08:04:24] Path Data                                logger=settings path=/var/lib/grafana
INFO[05-27|08:04:24] Path Logs                                logger=settings path=/var/log/grafana
INFO[05-27|08:04:24] Path Plugins                             logger=settings path=/var/lib/grafana/plugins
INFO[05-27|08:04:24] Path Provisioning                        logger=settings path=/etc/grafana/provisioning
INFO[05-27|08:04:24] App mode production                      logger=settings
INFO[05-27|08:04:24] Connecting to DB                         logger=sqlstore dbtype=sqlite3
WARN[05-27|08:04:24] SQLite database file has broader permissions than it should logger=sqlstore path=/var/lib/grafana/grafana.db mode=-rw-rw---- expected=-rw-r-----
INFO[05-27|08:04:24] Starting DB migrations                   logger=migrator
INFO[05-27|08:04:24] migrations completed                     logger=migrator performed=0 skipped=346 duration=464.523µs
INFO[05-27|08:04:24] Starting plugin search                   logger=plugins
INFO[05-27|08:04:24] Registering plugin                       logger=plugins id=input
INFO[05-27|08:04:24] Live Push Gateway initialization         logger=live.push_http
INFO[05-27|08:04:24] warming cache for startup                logger=ngalert
INFO[05-27|08:04:24] HTTP Server Listen                       logger=http.server address=[::]:3000 protocol=http subUrl= socket=
INFO[05-27|08:04:26] Request Completed                        logger=context userId=0 orgId=0 uname= method=GET path=/api/live/ws status=401 remote_addr=172.17.0.1 time_ms=0 size=27 referer=
```
> 这里的错误是正常的，因为没有对接 prometheus，也没有在 k8s 里面，收集不到这些数据。
![](/static/images/k8s/prometheus/grafana-migrate-alerts-rules.png)
![](/static/images/k8s/prometheus/grafana-migrate-1-alerts-rules.png)

---
# 问题汇总

## Grafana Logs "database is locked
[解决](https://github.com/grafana/grafana/issues/16638)

- 操作(2 选 1，建议 2)
    1.
    ```shell
    kubectl -n monitoring edit deployments.apps grafana
          - env:
            - name: GF_DATABASE_CACHE_MODE
              value: shared
    ```

    2.
    ```shell
    sqlite3 grafana.db 'pragma journal_mode=wal;'
    ```


## Templating Failed to upgrade legacy queries Datasource xxx not found

![](/static/images/k8s/prometheus/grafana-template-datasource-not-found.png)

## 参考
[Support dashboard variables in dashboard provisioning](https://github.com/grafana/grafana/issues/10786)

## 前言
> 从一套环境导出 dashboard 为 json 文件，在另一套环境中导入，那么出现如标题的问题, 应该怎么解决呢？1. 闲的蛋疼的话可以一个一个慢慢编辑，选择正确得 Datasource（日后导入到其他环境依旧需要如此操作）; 2. 直接编辑 json 文件（一劳永逸，环境随便换）

## 实操
1. 方式1
> 添加一个输入参数

```json
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "prometheus",
      "description": "",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
..................................................
        # 所有的这些
        "datasource": {
          "type": "prometheus",
          "uid": "xxxx-uid"
        },
        # 修改为
        "datasource": "${DS_PROMETHEUS}",
}
```
1. 方式2
> 硬编码
```json
{
  ...........................
        # 所有的这些
        "datasource": {
          "type": "prometheus",
          "uid": "xxxx-uid"
        },
        # 修改为
        # 所有的这些
        "datasource": {
          "type": "prometheus",
          "uid": "prometheus"
        },
  ...........................
}
```

1. 方式3
> 添加一个变量
```json
{
  .................
  # 找到 templating.list, 添加一个变量，这里的变量名为 DS_PROMETHEUS,值为 prometheus
  "templating": {
    "list": [
      {
        "datasource": "Prometheus",
        "description": null,
        "error": null,
        "hide": 0,
        "includeAll": false,
        "label": "datasource",
        "multi": false,
        "name": "DS_PROMETHEUS",
        "options": [],
        "query": "prometheus",
        "refresh": 1,
        "regex": "",
        "skipUrlSync": false,
        "type": "datasource"
      },
...
        # 所有的这些
        "datasource": {
          "type": "prometheus",
          "uid": "xxxx-uid"
        },
        # 修改为
        # 所有的这些
        "datasource": {
          "type": "prometheus",
          "uid": "${DS_PROMETHEUS}"
        },
}
```