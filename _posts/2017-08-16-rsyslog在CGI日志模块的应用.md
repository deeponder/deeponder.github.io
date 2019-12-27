---
layout:     post
title:      rsyslog在CGI日志模块的应用
subtitle:   
date:       2017-08-16
author:     jabin
header-img: 
catalog: true
tags:
    - 计算机
    - rsyslog
    - syslog
---

本文只介绍rsyslgo在CGI日志模块中的安装、常用命令和配置
# 背景
1. 公司的tlinux自带rsyslog， 而非syslog-ng
2. syslog-ng高版本安装较复杂，并且版本间的配置差异大，不方便部署
3. rsyslog采用多线程处理日志，性能上有较大提升

# 安装
1. tlinux自带rsyslog, 无需另外安装
2. 若系统已安装syslog-ng，可执行以下命令进行安装替换，这里注意卸载syslog-ng的时候，会连crontab一起卸载，因此需要重新安装crontab： 
`yum clean all ;yum install epel-release -y;yum remove syslog-ng -y; yum install rsyslog -y;yum install sysstat -y;yum install crontabs -y;/etc/init.d/crond restart;service rsyslog start`

# 常用命令
1. 启动/重启/停止：`service rsyslog start|restart|stop` 
2. 查看版本：`/sbin/rsyslog -v`
3. 测试配置文件是否正确：`rsyslogd -N1 -f file`

# rsyslog消息流处理
日志消息首先经过输入模块进入主队列，然后通过解析/过滤模块分解到各分队列，最后交由输出模块将日志输出各目标位置。

![我是图片](https://deeponder.github.io/img/tapd_1000046_1501298027_22.png)

# rsyslog配置
rsyslog的配置文件主要是配置`filter`和`action`， 并且还涉及模板的使用，属性的替换，以及全局指令和模块的加载。
日志文件的主体配置一般可以抽象为：`filter  action`， 表示日志消息经过过滤规则过滤后，交由指定的动作处理（或打到本地的日志文件，或通过udp/tcp打到远端机器）
## filter
### facility(产生日志的子系统).priority(日志级别)
facility: 包括 auth , authpriv , cron , daemon , kern , lpr , mail , news , syslog , user , ftp , uucp , local0 ~ local7
priority: 包括debug (7) , info (6) , notice (5) , warning (4) , err (3) , crit (2) , alert (1) , emerg (0) 
示例：
`cron.*`: 过滤出所有cron日志
`mail.crit`: 过滤出mail级别大于crit的日志
### 基于属性的过滤
语法：`:property, [!]compare-operation, "value"`
property： 属性相关后面会提到
compare-operation: 包括contains, isequal, startwith, regrex（正则）
示例：
`:msg, startswith, "hello you"`:  过滤出以"hello world"开头的日志
`:msg, contains, "ERR"`:  过滤出包含"ERR"的日志
###  基于表达式的过滤
语法：`if [conditions] then [actions] `
示例：
`if $programname startswith 'CGISYSLOG' then @10.166.135.83:514`: 过滤出`syslogtag`以"CGISYSLOG"开头的日志, 将其打到远程机器

## action
配置rsyslog系统在过滤日志后执行的操作
### 输出到静态日志文件
一般系统默认的日志（cron, kern等）是通过这种方式来打到固定的日志文件的
语法：`FILTER PATH`
示例：`*.info;mail.none;authpriv.none;cron.none  /var/log/messages`
### 输出到动态日志文件
CGI日志可以通过这种方式打到预定的模板路径
语法：`FILTER ?DymanicFilePath`
示例:
```
$template DefalutLogFormat, "%$NOW% %TIMESTAMP:8:15%|%fromhost-ip%|%msg:2:$:::drop-last-lf%\n"                 ---日志格式模板
$template d_weblog_local, "/data/qqlog/WEBLOG/%$year%%$month%/%programname%-%$year%%$month%%$day%-%$hour%.log"    ---日志文件路径模板
if $programname startswith 'WEBLOG' then ?d_weblog_local;DefalutLogFormat
```
### 输出到远程机器
场景：外网众多的机器，有时我们需要将一些重要的账单日志打到统一的一台机器，方便查询
语法（注：zNUMBER定义是否对日志进行zlib压缩（级别1~9））： 

1. udp: `@[(zNUMBER)][ip]:[PORT]`
2. tcp: `@@[(zNUMBER)]HOST:[PORT]`  

示例：

tips: 若需要同时把过滤后的日志打到本地和远程机器可以使用如下配置：
```

```

### 丢弃日志
语法： `FILTER ~`
示例：`& ~`: 后面所有的日志均丢弃

## 属性
属性列表如下(红色为常用属性):
|    属性名    | 说明 |
| ---------- | --- |
|  !!#ff0000 msg!! | 日志正文|
|hostname| 日志中的主机名|
|fromhost| 从该主机接收到的消息，可能不是最开始的发送主机|
|fromhost-ip| fromhost 的IP|
|syslogtag| 日志标签，如 named[12345]|
| !!#ff0000 programname!! | 日志标签的静态部分，如 named|
|pri| 日志的 PRI 部分|
|pri-text| PRI| 的文本表示，如 syslog.info|
|syslogfacility| 日志类别|
|syslogfacility-text| 日志类别的文本表示|
|syslogseverity| 日志级别|
|syslogseverity-text| 日志级别的文本表示|
|timegenerated| 日志接收时间，或理解为 timereceived|
|timereported| 日志内的报告时间，或生成时间|
|$ !!#ff0000 now!! | 当前时间，YYYY-MM-DD|
|$ !!#ff0000 year!! | 当前年，YYYY|
|$ !!#ff0000 month!! | 当前月，MM|
|$ !!#ff0000 day!! | 当前日志，DD|
|$ !!#ff0000 hour!! | 当前小时，24 小时格式，HH|
|$ !!#ff0000 hhour!! | 当前半小时，0-29 对应 0，30-59| 对应| 1|
|$ !!#ff0000 qhour!! | 当前1/4小时，0-3|
|$ !!#ff0000 minute!! | 当前分钟，MM|
tips: 属性可以配合模板进行属性值替换
## 模板
语法(注意，这里的PROPERTY即上面提到的属性)：`$template TEMPLATE_NAME,"text %PROPERTY% more text"`
示例：
`$template d_weblog_local, "/data/qqlog/WEBLOG/%$year%%$month%/%programname%-%$year%%$month%%$day%-%$hour%.log"`

## 全局指令和模块加载
全局指令：rsyslogd守护进程的配置指令。如设置消息大小：`$MaxMessageSize 20k`
模块加载：加载输入模块，udp模块等。 如`$ModLoad imuxsock.so` (注：imuxsock: Unix Socket Input）

# 最后
目前actc.minigame.qq.com的日志系统已经替换为rsyslog，经过一周的日志观察，运行良好。相关的配置参考附件`rsyslog.conf`。 
