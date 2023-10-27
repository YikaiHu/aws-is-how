# Fluent-bit Cheat Sheet

### 如何启动flb

ec2: 
```
/opt/fluent-bit/bin/fluent-bit -c /opt/fluent-bit/etc/fluent-bit.conf
```
ecs: 
```
/fluent-bit/bin/fluent-bit -c /fluent-bit/etc/fluent-bit.conf
```

### 如何EC2 Debug
```
journalctl -u fluent-bit.service
```

### 如何输出到stdout
```
[OUTPUT]
    Match *
    Name                stdout
    Format              json_lines
    json_date_format    iso8601
    json_date_key       sys_time
```

### 查看Fluent-bit的数据发送情况
```
curl -s http://127.0.0.1:2022/api/v1/metrics | jq

curl -s http://127.0.0.1:2022/api/v1/metrics/prometheus
```

### 如何检查输入的原始log
```
[PARSER]
    Name           parser_dev
    Format         regex
    Regex          ^(?<raw_message>.*)$
```

### 如何安装的Fluent-bit 社区版
```
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
```

### 快速调试Fluent-bit的时间戳
```
cd /opt/fluent-bit
mkdir etc
cd etc
vi fluent-bit.conf
```
贴入以下内容
```
[SERVICE]
    Flush                       5
    Daemon                      off
    Log_level                   Info
    Http_server                 On
    Http_listen                 0.0.0.0
    Http_port                   2022
    log_File                    /tmp/log-agent.log
    storage.path                /opt/fluent-bit/flb-storage/
    storage.sync                normal
    storage.checksum            off
    storage.backlog.mem_limit   5M
    Parsers_File                /opt/fluent-bit/etc/applog_parsers.conf

[INPUT]
    Name                        tail
    Path                        /tmp/dev.log
    Path_Key                    file_name
    Tag                         dev-001
    Read_from_head              true
    Mem_Buf_Limit               30M
    storage.type                filesystem
    Buffer_Chunk_Size           512k
    Buffer_Max_Size             5M
    Parser                      regex_dev_parser_01



[OUTPUT]
    Match *
    Name                stdout
    Format              json_lines
    json_date_format    iso8601
    json_date_key       sys_time
```
创建parser
```
vi applog_parsers.conf
```
```
[PARSER]
    Name        regex_dev_parser_01
    Format      regex
    Time_Key    time
    Regex       ^\<(?<pri>[0-9]{1,5})\>1 (?<time>[^ ]+) (?<host>[^ ]+) (?<ident>[^ ]+) (?<pid>[-0-9]+) (?<msgid>[^ ]+) (?<extradata>(\[(.*)\]|-)) (?<message>.+)$
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```