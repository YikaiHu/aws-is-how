# Centralized Logging with OpenSearch Workshop CN version

Click the template link, and download the template.

- [CloudFormation template for cn-north-1](https://github.com/YikaiHu/aws-is-how/blob/main/tools/clo-workshop-cn/CLWorkshopEC2AndEKS-cn-north-1.template)
- [CloudFormation template for cn-northwest-1](https://github.com/YikaiHu/aws-is-how/blob/main/tools/clo-workshop-cn/CLWorkshopEC2AndEKS-cn-northwest-1.template)

Go to CloudFormation Console and deploy the stack by upload the template yaml.

You can visit the e-commerce website through the alb link in CloudFormation Output.

> **Note**
> If you are unable to access this interface, please ensure that your AWS China account has completed the ICP whitelist filing and has opened ports 80 and 443.
> Please refer to this [guide](http://cet-bucket.s3.cn-north-1.amazonaws.com.cn/Process/ICP%20Exception%20Request/BMS%20ICP%20Exception%20Request%20%E6%93%8D%E4%BD%9C%E8%AF%B4%E6%98%8E.pdf).

![eks-website](./eks-website.png)

## How to crete application log in workshop EKS cluster - DaemonSet

1. 在本地 Vscode 中关联 EKS
2. 下载 [clo-eks-uat.yaml](./clo-eks-uat.yaml)
3. 执行`kubectl apply -f clo-eks-uat.yaml`, 你可以看到如下的pods
![eks-flog-pods](./eks-flog-pods.png)

其中json 日志还有 apache 日志来自于flog，其余的复用workshop的电商网站里面的pod.

* [Nginx Log](#nginx-log)
* [Apache Log](#apache-log)
* [Json Log](#json-log)
* [Single-line Log](#single-line-log)
* [Spring-boot Log](#spring-boot-log)

### Nginx Log
- Log format:
  ```
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';
  ```

- Sample log:
  ```
  127.0.0.1 - - [24/Dec/2021:01:27:11 +0000] "GET / HTTP/1.1" 200 3520 "-" "curl/7.79.1" "-"
  ```

- Log path: 
  ```
  /var/log/containers/nginx-svc*
  ```

### Apache Log
- Log format:
  ```
  LogFormat "%h %l %u %t \"%r\" %>s %b" combined
  ```

- Sample log:
  ```
  55.146.151.251 - - [24/Jul/2022:07:59:59 +0000] "POST /e-enable HTTP/1.1" 405 13357
  ```

- Log path: 
  ```
  /var/log/containers/flog-apache*.log
  ```

### Json Log

- Sample log:
  ```
    {"host":"176.54.164.169", "user-identifier":"kling1353", "time":"24/Jul/2022:07:56:40 +0000", "method": "PUT", "request": "/unleash/magnetic/matrix", "protocol":"HTTP/1.1", "status":403, "bytes":29229, "referer": "http://www.chiefleading-edge.info/frictionless"}
  ```
- Log time format:
  ```
  %d/%b/%Y:%H:%M:%S %z
  ```
- Log path: 
  ```
  /var/log/containers/flog-json*.log
  ```

### Single-line Log
- Regex:
  ```
  (?<remote_addr>[0-9.-]+)\s+(?<remote_ident>[\w.-]+)\s+(?<remote_user>[\w.-]+)\s+\[(?<time_local>[^\[\]]+|-)\]\s+\"(?<request_method>(?:[^"]|\")+)\s(?<request_uri>(?:[^"]|\")+)\s(?<request_protocol>(?:[^"]|\")+)\"\s+(?<status>\d{3}|-)\s+(?<response_size_bytes>\d+|-).*
  ```

- Sample log:
  ```
  55.146.151.251 - - [24/Jul/2022:07:59:59 +0000] "POST /e-enable HTTP/1.1" 405 13357
  ```
- Log time format:
  ```
  %d/%b/%Y:%H:%M:%S %z
  ```
- Log path: 
  ```
  /var/log/containers/flog-apache*.log
  ```

### Spring-boot Log
- Log format:
  ```
  %d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger : %msg%n
  ```

- Sample log:
  ```
  2022-02-18 10:32:26 ERROR [http-nio-8080-exec-1] org.apache.catalina.core.ContainerBase.[Tomcat].[localhost].[/].[dispatcherServlet] : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.ArithmeticException: / by zero] with root cause
  java.lang.ArithmeticException: / by zero
    at com.springexamples.demo.web.LoggerController.logs(LoggerController.java:22)
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke
  ```

- Log path: 
  ```
  /var/log/containers/spring-boot*
  ```

## How to crete application log in workshop EKS cluster - Sidecar

需要拼接一下不同Log Generator Container 和CLO生成的sidecar sh，并且mount一下path
案例如下

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: logging

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
  annotations:
    eks.amazonaws.com/role-arn: arn:aws-cn:iam::123456789012:role/LogHub-EKS-LogAgent-Role-6b72c1ba1413488aa9d533edd7fb6245

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-read
rules:
  - nonResourceURLs:
      - /metrics
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - namespaces
      - pods
      - pods/logs
      - nodes
      - nodes/proxy
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-read
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-read
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
  labels:
    k8s-app: fluent-bit
data:
  # Configuration files: server, input, filters and output
  # ======================================================
  uniform-time-format.lua: |
    function cb_print(tag, timestamp, record)
        record['time'] = string.format(
            '%s.%sZ',
            os.date('%Y-%m-%dT%H:%M:%S', timestamp['sec']),
            string.sub(string.format('%06d', timestamp['nsec']), 1, 6)
        )
        return 2, timestamp, record
    end
    
  fluent-bit.conf: |
    [SERVICE]
        Flush                       5
        Daemon                      off
        Log_level                   Info
        Http_server                 On
        Http_listen                 0.0.0.0
        Http_port                   2022
        Parsers_File                parsers.conf
        storage.path                /var/fluent-bit/state/flb-storage/
        storage.sync                normal
        storage.checksum            off
        storage.backlog.mem_limit   5M
    
    [INPUT]
        Name                tail
        Tag                 loghub.e4051755-482c-47c0-9108-028e496ceca0.a0999536-a5a5-416d-ba4f-025583a9a5c9.*
        Path                /var/log/spring-boot/access.log
        Path_Key            file_name
        DB                  /var/fluent-bit/state/flb_container-e4051755-482c-47c0-9108-028e496ceca0.a0999536-a5a5-416d-ba4f-025583a9a5c9.db
        DB.locking          True    
        Multiline           On
        Parser_Firstline    java_spring_boot_e4051755-482c-47c0-9108-028e496ceca0
        Mem_Buf_Limit       50MB
        # Since "Skip_Long_Lines" is set to "On", be sure to adjust the "Buffer_Chunk_Size","Buffer_Max_Size" according to the actual log. If the parameters are adjusted too much, the number of duplicate records will increase. If the value is too small, data will be lost. 
        # https://docs.fluentbit.io/manual/pipeline/inputs/tail
        Buffer_Chunk_Size   32k
        Buffer_Max_Size     128K
        Skip_Long_Lines     On
        Skip_Empty_Lines    On
        storage.type        filesystem
        Read_from_Head      True

    [OUTPUT]
        Name                kinesis_streams
        Match               loghub.e4051755-482c-47c0-9108-028e496ceca0.a0999536-a5a5-416d-ba4f-025583a9a5c9.*
        Region              cn-northwest-1
        Stream              LogHub-EKS-Cluster-PodLog-Pipeline-a0999-Stream790BDEE4-bR4VzxJqfdKS
        Retry_Limit         False
        Role_arn            arn:aws-cn:iam::379076500825:role/LogHub-EKS-Cluster-PodLog-DataBufferKDSRole7BCBC83-FUSGA9R45JSH


    [FILTER]
        Name                modify
        Match               loghub.*
        Set                 cluster ${CLUSTER_NAME}

    [FILTER]
        Name                lua
        Match               loghub.*
        time_as_table       on
        script              uniform-time-format.lua
        call                cb_print
    

  parsers.conf: |
    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
        Name         docker
        Format       json
        Time_Key     container_log_time
        Time_Format  %Y-%m-%dT%H:%M:%S.%LZ
        Time_Keep    On

    [PARSER]
        Name        cri_regex
        Format      regex
        Regex       ^(?<container_log_time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$      
        Time_Key    container_log_time
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ
        Time_Keep    On        


    [PARSER]
        Name        java_spring_boot_e4051755-482c-47c0-9108-028e496ceca0
        Format      regex
        Regex       (?<time>\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2})\s+(?<level>[\S]+)\s+\[(?<thread>.+)\]\s+(?<logger>\S+)\s+:\s+(?<message>[\s\S]+)
        Time_Key    time
        Time_Format %Y-%m-%d %H:%M:%S



---
apiVersion: v1
kind: Pod
metadata:
  namespace: logging
  name: app-sidecar
  labels:
    app: app-sidecar
spec: 
  containers:
  - name: spring-boot
    image: openjdk:11-jre
    volumeMounts:
      - name: app-log
        mountPath: /var/log/spring-boot  
    resources:
      limits:
        memory: 512Mi
    args:
      - /bin/bash
      - -xec
      - |
        cd /tmp
        wget https://aws-gcr-solutions.s3.amazonaws.com/log-hub-workshop/v1.0.0/petstore-0.0.1-SNAPSHOT.jar
        mkdir -p /var/log/spring-boot/
        java -jar petstore-0.0.1-SNAPSHOT.jar --server.port=80 | tee /var/log/spring-boot/access.log
    ports:
      - containerPort: 80
        protocol: TCP
  - name: ping-spring-boot
    image: badouralix/curl-jq
    resources:
      limits:
        memory: 512Mi
    args:
      - /bin/sh
      - -ec
      - |
        curl -LSs "https://hub.fastgit.xyz/wchaws/flog/releases/download/v0.5.0-20220529/flog_0.5.0-20220529_linux_arm64.tar.gz" -o /tmp/flog.tar.gz
        cd /tmp
        tar -xvzf flog.tar.gz
        mv flog /usr/bin
        while true; do
          flog -f json -n 10 | jq -r '"http://localhost" + .request' | xargs -I {} sh -c "curl {} -o /dev/null; sleep 1;" &> /dev/null
          curl http://localhost/hello &> /dev/null
        done
  # Fluent-bit's container
  - name: fluent-bit
    image: public.ecr.aws/aws-observability/aws-for-fluent-bit:stable
    imagePullPolicy: Always
    env:
      - name: CLUSTER_NAME
        value: "loghub-eks-1659346435"
    ports:
      - containerPort: 2022
    resources:
        limits:
          memory: 200Mi
        requests:
          memory: 100Mi
    volumeMounts:
    - name: var-log
      mountPath: /var/log
    - name: var-lib-docker-containers
      mountPath: /var/lib/docker/containers
      readOnly: true
    - name: app-log
      mountPath: /var/log/spring-boot  
    - name: fluentbitstate
      mountPath: /var/fluent-bit/state  
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc/ 
      readOnly: true    
  volumes:
  - name: var-log
    hostPath:
      path: /var/log
  - name: var-lib-docker-containers
    hostPath:
      path: /var/lib/docker/containers      
  - name: app-log
    emptyDir: {}
  - name: fluentbitstate
    hostPath:
      path: /var/fluent-bit/state      
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config   
  serviceAccountName: fluent-bit   
```