---
title: "Docker 配置 Prometheus+Alertmanager+Grafana+Mysql"
date: 2023-02-28
lastmod: 2023-02-28
draft: false
tags: ["note"]
categories: ["Note"]
authors:
  - Ash
---

# Docker 配置 Prometheus+Alertmanager+Grafana+Mysql

## Prometheus

1、下载镜像包：

```shell
docker pull prom/prometheus
```

2、配置 Prometheus

```shell
mkdir /opt/prometheus
cd /opt/prometheus/
vim prometheus.yml

#opt是linux中存放第三方软件的文件夹
```

内容如下：

```shell
global:
  scrape_interval:     60s
  evaluation_interval: 60s


scrape_configs:

  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  - job_name: Data
    static_configs:
      - targets: ['192.168.37.184:5000']	#python中对应的端口号是5000，所以从主机5000端口接收数据
        labels:
          instance: Data

alerting:
  alertmanagers:
    - static_configs:
      - targets: ['172.17.0.1:9093']	#这里的ip是docker的ip地址，也可以改为宿主机的ip地址，但对应暴露的端口号也要改变

rule_files:
  - rules.yml


```

3、配置 rules.yml，即 alert 的规则

```shell
vim /opt/prometheus/rules.yml
```

内容如下：

```yaml
groups:
  - name: server-alarm
    rules:
      - alert: "内存告警"
        expr: (1 - (node_memory_MemAvailable_bytes / (node_memory_MemTotal_bytes))) * 100 > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: 检测到 高内存 使用率！"
          description: "{{$labels.instance}}: 内存使用率在 80% 以上 (当前使用值为:{{ $value }})"

      - alert: "CPU告警"
        expr: (1 - avg(irate(node_cpu_seconds_total{mode="idle"}[2m])) by(instance)) * 100 > 80
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: 检测到 高CPU 使用率！"
          description: "{{$labels.instance}}: CPU使用率在 80% 以上 (当前使用值为:{{ $value }})"

      - alert: "磁盘告警"
        expr: 100 - (node_filesystem_free_bytes{fstype=~"tmpfs|ext4"} / node_filesystem_size_bytes{fstype=~"tmpfs|ext4"} * 100) > 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{$labels.instance}}: 检测到 高磁盘 使用率！"
          description: "{{$labels.instance}}: 磁盘使用率在 80% 以上 (当前使用值为:{{ $value }})"

      - alert: "InstanceUp"
        expr: up == 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }}"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been up for more than 1 minutes."
```

4、网络准备

因为需要在本地运行两个容器，且两个容器需要互相能否访问，所以这里创建一个 network。

```shell
docker network create prom
```

查看：

```shell
[root@centos01 ~]# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
086bf47d28da   bridge    bridge    local
19357bf4da6f   host      host      local
14df82a9ab10   none      null      local
e09f984b92bc   prom      bridge    local

[root@centos01 ~]# docker inspect e09f984b92bc
[
    {
        "Name": "prom",
        "Id": "e09f984b92bc77fcaa9b5c6b1090938dde918f739b5eb67a18e56eb1f8c4eead",
        "Created": "2022-07-13T15:02:27.09181599+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

5、启动 Prometheus

```shell
docker run  -d --name prometheus --restart=always -p 9090:9090 -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -v  /opt/prometheus/rules.yml:/etc/prometheus/rules.yml  prom/prometheus
```

## AlertManager

### 环境配置及运行

1、准备镜像

```shell
docker pull prom/alertmanager
```

2、Alertmanager 配置：

- 先用 docker 将其运行起来

```shell
docker run -d --name alertmanager --network prom -p 9093:9093 prom/alertmanager
```

- 再将容器中的的 alertmanager 文件夹复制到宿主机内的/opt/alertmanager 中

```shell
docker cp alertmanager:/etc/alertmanager /opt/alertmanager
```

- 修改 alertmanager.yml 配置文件

```shell
vim /opt/alertmanager/alertmanager.yml
```

如下：
[alertmanager 配置详解](https://blog.csdn.net/yuezhilangniao/article/details/119766157)
match_re 可以用正则表达式匹配

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: "xxxx"
  smtp_from: "xxxx"
  smtp_auth_username: "xxxx"
  smtp_auth_password: "xxxx"
  smtp_require_tls: false

route:
  group_by: ["IP", "alertname"]
  group_wait: 20s
  group_interval: 5m
  repeat_interval: 30m
  receiver: Test
  routes:
    - match:
        IP: xxxx|xxxx
      receiver: Tom
    - match:
        IP: xxxx|xxxx|xxxx
      receiver: Corey
    - match:
        IP: xxxx
      receiver: Ash
    - match_re:
        system_name: ".*Kirk.*"
      receiver: Kirk

receivers:
  - name: "Tom"
    email_configs:
      - to: "123@qq.com"
        send_resolved: true

  - name: "Corey"
    email_configs:
      - to: "222@qq.com"
        send_resolved: true

  - name: "Ash"
    email_configs:
      - to: "333@qq.com"
        send_resolved: true

  - name: "Kirk"
    email_configs:
      - to: "444@qq.com"
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["alertname", "dev", "instance"]
```

- 把宿主机的文件夹挂载到容器上，再运行，这样修改宿主机的 yml 文件就能容器内文件也对应改变

```shell
docker run -itd --name alertmanager -p 9093:9093 -v /opt/alertmanager:/etc/alertmanager prom/alertmanager
```

## Grafana

1、拉取镜像

```shell
docker pull grafana/grafana
```

新建空文件夹 grafana-storage，用来存储数据

```shell
mkdir /opt/grafana-storage
```

2、添加权限

```shell
chmod 777 -R /opt/grafana-storage
```

3、启动 grafana

```shell
docker run -d --name grafana --restart=always -p 521:3000 --name=grafana -v /opt/grafana-storage:/var/lib/grafana grafana/grafana
```

4、访问 url：

http://ip:521/

5、登陆

默认的用户密码都是：admin

### 移植

将/var/lib/grafana 下的：

- plugins 文件夹
- grafana.db 文件

复制到新的 grafana 上面就行

## Mysql

### 安装和配置

1、安装 mysql

```shell
docker pull mysql
```

2、创建必要的文件夹

```shell
cd /opt # 我的软件都安装在/opt目录下。
mkdir mysql_docker
cd mysql_docker/
echo $PWD # 当前位置的绝对路径
```

3、启动 mysql 容器

```shell
docker run --name mysqlserver --restart=always --privileged=true -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d -i -p 3306:3306 mysql:latest

# $PWD是当前位置的绝对路径，所以必须在2中创建的文件夹下执行该命令
# --privileged=true：在docker run中加入 --privileged=true 给容器加上特定权限
# --name mysqlserver :指定容器的名字为mysqlserver
# -v $PWD/conf:/etc/mysql/conf.d：将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。
# -v $PWD/data:/var/lib/mysql ：将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。
# -e MYSQL_ROOT_PASSWORD=123456：初始化 root 用户的密码。
```

4、测试 mysql 登录

```shell
docker exec -it mysqlserver /bin/bash # 也可以把容器名"mysqlserver"改为容器id
mysql -uroot -p
#设置了初始化密码是123456
```

5、测试远程连接

```shell
firewall-cmd --zone=public --add-port=3306/tcp --permanent # 开放3306端口
firewall-cmd --reload # 配置立即生效
firewall-cmd --zone=public --list-ports
```

### 复制 mysql（本地到虚拟机）

1、在虚拟机创建一个数据库用来导入（最好数据库名字与本机中的相同）

2、从虚拟机登录本地 mysql

```shell
mysql -uroot -p -h192.168.37.184 -P3306	#这一步是进入mysql的，可以不做，直接到第三步
```

3、备份本地数据库到虚拟机

**以下都是在虚拟机 mysql 命令行执行，不在登录后的"mysql>"中执行**

- 导出：连接本地 mysql，导出数据库

```shell
mysqldump -h192.168.37.184 -P3306 -uroot -proot monitor > bak.sql;
#格式：mysqldump -h链接ip -P(大写)端口 -u用户名 -p密码 数据库名>d:XX.sql(路径)
```

- 导入：导入到虚拟机 mysql

```shell
mysql -uroot -ppassword monitor <bak.sql
```

**注意：-p 可以放在最后，因为密码显式输入会报 warning**

### 遇到的问题

- Host \* is not allowed to connect to this MySQL server（多次登陆失败）

```shell
#在要访问的那个远程mysql中执行以下命令
mysql -u root -p

mysql>use mysql;

mysql>update user set host ='%'where user ='root';

mysql>flush privileges;
```

# Docker 遇到的问题

- Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

```shell
service docker restart
```

- Prometheus 时间不同步

```shell
ntpdate ntp.aliyun.com
```
