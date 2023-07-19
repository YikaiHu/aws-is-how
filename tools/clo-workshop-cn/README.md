## Centralized Logging with OpenSearch Workshop CN version

Click the template link, and download the template.

- [CloudFormation template for cn-north-1](https://github.com/YikaiHu/aws-is-how/blob/main/tools/clo-workshop-cn/CLWorkshopEC2AndEKS-cn-north-1.template)
- [CloudFormation template for cn-northwest-1](https://github.com/YikaiHu/aws-is-how/blob/main/tools/clo-workshop-cn/CLWorkshopEC2AndEKS-cn-northwest-1.template)

Go to CloudFormation Console and deploy the stack by upload the template yaml.

You can visit the e-commerce website through the alb link in CloudFormation Output.

> **Note**
  If you are unable to access this interface, please ensure that your AWS China account has completed the ICP whitelist filing and has opened ports 80 and 443.
  Please refer to this [guide](http://cet-bucket.s3.cn-north-1.amazonaws.com.cn/Process/ICP%20Exception%20Request/BMS%20ICP%20Exception%20Request%20%E6%93%8D%E4%BD%9C%E8%AF%B4%E6%98%8E.pdf).

![eks-website](./eks-website.png)