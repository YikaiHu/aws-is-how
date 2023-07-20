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

## How to crete application log in workshop EKS cluster

1. 在本地 Vscode 中关联 EKS
2. 下载 [clo-eks-uat.yaml](./clo-eks-uat.yaml)
3. 执行`kubectl apply -f clo-eks-uat.yaml`, 你可以看到如下的pods
![eks-flog-pods](./eks-flog-pods.png)

其中json 日志还有 apache 日志来自于flog，其余的复用workshop的电商网站里面的pod.

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