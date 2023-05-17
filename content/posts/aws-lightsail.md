---
title: "Aws Lightsail 使用 Cloudwatch-agent 监控流量使用" 
date: 2023-05-17T20:54:49+08:00
# weight: 1
# aliases: ["/first"]
tags: ["linux","aws","cloudwatch"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
editPost:
    URL: "https://github.com/letterl/letterl.github.io/tree/main/content/posts"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
先上效果图
![](https://up-web.pages.dev/img/20230517212000.16d0bc8b.png "")



系统是Ubuntu

## 1. 安装Aws-cli

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```





## 2. 安装 Cloudwatch-agent

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

## 3. 配置 Cloudwatch-agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```


### 设置验证信息

获取验证信息 创建IAM用户时 创建编程访问密钥


```bash
sudo aws configure --profile AmazonCloudWatchAgent
```
access key和secret access key参照创建IAM用户时生成的csv文件输入。
输入前两项 其他默认
```bash
AWS Access Key ID [None]:******************XZ
AWS Secret Access Key [None]: ******************5a
Default region name [None]:
Default output format [None]:
```


### 生成 agent.json 这个是配置文件 定义收集系统哪些信息

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

>您使用的是 EC2 还是 On-Premises 主机？
>1. EC2
>2. On-Premises
>默认选择：[1]:
>2
>Do you want to turn on StatsD daemon?
>1. yes
>2. no
>默认选择：[1]:
>2
>Do you want to monitor metrics from CollectD?
>1. yes
>2. no
>默认选择：[1]:
>2
>
>您要监控每个核心的 CPU 指标吗？可能会收取额外的 CloudWatch 费用。1
>. 是
>2. 否
>默认选择：[1]:
>2
>如果信息可用，是否要将 ec2 维度（ImageId、InstanceId、InstanceType、AutoScalingGroupName）添加到所有指标中？1.
>是
>2. 无
>默认选择：[1]：
>2
>Do you want to monitor any log files?
>1. yes
>2. no
>默认选择：[1]:
>2
>Do you want to store the config in the SSM parameter store?
>1. yes
>2. no
>默认选择：[1]:
>2

### 设置 CloudWatch 代理
配置 CloudWatch 代理以使用之前设置的配置文件
```bash
sudo vim /opt/aws/amazon-cloudwatch-agent/etc/common-config.toml
```

添加以下内容
```bash
[credentials]
shared_credential_profile = "AmazonCloudWatchAgent"
```

启用代理
```bash
# 启动
sudo amazon-cloudwatch-agent-ctl -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -a fetch-config -s
# 状态
sudo amazon-cloudwatch-agent-ctl -a status
# 停止
sudo amazon-cloudwatch-agent-ctl -a stop
# 日志
sudo cat /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### 问题排查
```bash

sudo systemctl status amazon-cloudwatch-agent
● amazon-cloudwatch-agent.service - Amazon CloudWatch Agent
   Loaded: loaded (/etc/systemd/system/amazon-cloudwatch-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-09-29 14:36:42 UTC; 36s ago
 Main PID: 19055 (amazon-cloudwat)
    Tasks: 7
   Memory: 19.1M
   CGroup: /system.slice/amazon-cloudwatch-agent.service
           └─19055 /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent -config /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.toml -envconfig /opt/aws/amazon-cloudwatch-agent...

Sep 29 14:36:42 ip-172-26-0-182.ap-northeast-1.compute.internal systemd[1]: Stopped Amazon CloudWatch Agent.
Sep 29 14:36:42 ip-172-26-0-182.ap-northeast-1.compute.internal systemd[1]: Started Amazon CloudWatch Agent.
Sep 29 14:36:42 ip-172-26-0-182.ap-northeast-1.compute.internal start-amazon-cloudwatch-agent[19055]: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json does not exist or cann...ing it.
Sep 29 14:36:42 ip-172-26-0-182.ap-northeast-1.compute.internal start-amazon-cloudwatch-agent[19055]: I! Detecting run_as_user...
Hint: Some lines were ellipsized, use -l to show in full.
```
使用软链接修复

```bash
cd /opt/aws/amazon-cloudwatch-agent/etc/
sudo ln -s ../bin/config.json amazon-cloudwatch-agent.json
ls -l /opt/aws/amazon-cloudwatch-agent/etc/
```

贴一下我的amazon-cloudwatch-agent.json
```json
    {
        "agent": {
                "metrics_collection_interval": 60,
                "run_as_user": "root"
        },
        "logs": {
                "logs_collected": {
                        "files": {
                                "collect_list": [
                                        {
                                                "file_path": "/var/log/messages",
                                                "log_group_name": "syslog",
                                                "log_stream_name": "{instance_id}",
                                                "retention_in_days": -1
                                        }
                                ]
                        }
                }
        },
        "metrics": {
                "aggregation_dimensions": [
                        [
                                "InstanceId"
                        ]
                ],
                "append_dimensions": {
                        "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
                        "ImageId": "${aws:ImageId}",
                        "InstanceId": "${aws:InstanceId}",
                        "InstanceType": "${aws:InstanceType}"
                },
                "metrics_collected": {
                        "cpu": {
                                "measurement": [
                                        "cpu_usage_idle",
                                        "cpu_usage_iowait",
                                        "cpu_usage_user",
                                        "cpu_usage_system"
                                ],
                                "metrics_collection_interval": 60,
                                "resources": [
                                        "*"
                                ],
                        "totalcpu": false
                        },
                        "disk": {
                                "measurement": [
                                        "used_percent"
                                ],
                                "metrics_collection_interval": 60,
                                "resources": [
                                        "*"
                                ]
                        },
                        "mem": {
                                "measurement": [
                                        "mem_used_percent"
                                ],
                                "metrics_collection_interval": 60
                        },
                        "netstat": {
                                "measurement": [
                                        "tcp_established",
                                        "tcp_time_wait"
                                ],
                                "metrics_collection_interval": 60
                        },
                        "statsd": {
                                "metrics_aggregation_interval": 60,
                                "metrics_collection_interval": 10,
                                "service_address": ":8125"
                        },
                              "net": {
                                  "measurement": [
                                        "bytes_sent",
                                         "bytes_recv",
                                         "drop_in",
                                         "drop_out","err_in","err_out","packets_sent","packets_recv"],
                                "metrics_collection_interval": 60,
                                "unit":"Gigabytes"
                         }
                }
        }
}
```

最后 使用非主账号也就是非IAM账号登入，然后在CloudWatch中查看监控数据
在全部指标 选项中 选择lightsail实例的地区 选择CWAgent
添加自己需要的监控指标即可