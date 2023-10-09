# How to create 50 EC2 at once

Create a shell script.

```shell
#!/bin/bash

INSTANCE_COUNT=70

IMAGE_ID="resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
INSTANCE_TYPE="t2.medium"
KEYPAIR="YOUR_KEY_NAME"
TAG_NAME="run-ec2-amazon-linux2"
IAM_ROLE_NAME="YOUR_EC2_ROLE_NAME" 

for ((i=1; i<=$INSTANCE_COUNT; i++))
do
  INSTANCE_NAME="${TAG_NAME}-${i}"
  echo "Creating ${INSTANCE_NAME} instance..."
  aws ec2 run-instances \
    --image-id $IMAGE_ID \
    --instance-type $INSTANCE_TYPE \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${INSTANCE_NAME}}]" \
    --iam-instance-profile Name=$IAM_ROLE_NAME \
    --user-data "#!/bin/bash -ex
      exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
      sudo yum update -y
      sudo yum install -y openssl11
      sudo amazon-linux-extras install -y nginx1 java-openjdk11 || sudo amazon-linux-extras install -y nginx1 java-openjdk11   # double install to make sure installation is successful
    " \
    --key-name $KEYPAIR --query 'Instances[].InstanceId' --output text >> instance-ids.txt
    
done

echo "EC2 instances creation complete."
```

Give the script the permission.

```
chmod +x launch-ec2.sh
```