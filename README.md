# EMR Hands-on Workshop

### Section 1
Objective : Understand the configuration options in creating a EMR cluster using AWS CLI

#1. Our VPC will be a very small range /24, with our subnet space using a /28 netmask.
```sh
aws ec2 create-vpc --cidr-block 10.20.30.0/24 --instance-tenancy default
aws ec2 create-tags --resources vpc-0465fe8ce0f1f0af9 --tags Key=Name,Value=vpc-emr-handson
```

#2. Let’s create a subnet to go along with the VPC, and specify the VPC ID and the range for the subnet. In this example, we will use 10.20.30.0/28.
```sh
aws ec2 create-subnet --vpc-id vpc-0465fe8ce0f1f0af9 --cidr-block 10.20.30.0/28
aws ec2 create-tags --resources subnet-071ed66bc1722e8b5 --tags Key=Name,Value=subnet-emr-handson
```

#3. We need a route table with a public Internet gateway. We will issue the create-route-table command with the VPC ID from earlier. Next, we will create the default route with the create-route command.
```sh
aws ec2 create-route-table --vpc-id vpc-0465fe8ce0f1f0af9
aws ec2 create-tags --resources rtb-0275c8f833b6119e2 --tags Key=Name,Value=rtb-emr-handson
```

#4. Because it is required by EMR, we will create a new internet gateway.
```sh
aws ec2 create-internet-gateway
aws ec2 create-tags --resources igw-08762ab2d6842477d --tags Key=Name,Value=igw-emr-handson
```

#5. Next, we attach the Internet gateway to the VPC:
```
aws ec2 attach-internet-gateway --internet-gateway-id igw-08762ab2d6842477d --vpc-id vpc-0465fe8ce0f1f0af9
```

#6. We will make the Internet gateway the default route using the Internet gateway ID and route table ID from earlier.
```sh
aws ec2 create-route --route-table-id rtb-0275c8f833b6119e2 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-08762ab2d6842477d
```

#7. And attach the route table:
```sh
aws ec2 associate-route-table --route-table-id rtb-0275c8f833b6119e2 --subnet-id subnet-071ed66bc1722e8b5
```

#8. We need to check to see if DNS hostnames are enabled. If they are not, we will enable them.
```sh
aws ec2 describe-vpc-attribute  --vpc-id vpc-055ef660 --attribute  enableDnsHostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-055ef660 --enable-dns-hostnames
```

#9. Finally, we have everything in place to launch a cluster inside of the VPC successfully. We can use the following command to launch a test cluster
```sh
aws emr create-cluster \
   --name emr-handson-cluster \
   --emrfs Consistent=true \
   --use-default-roles \
   --applications Name=Spark \
   --ec2-attributes KeyName=ea-keypair,SubnetId=subnet-071ed66bc1722e8b5 \
   --release-label emr-6.1.0 \
   --instance-groups InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m3.xlarge InstanceGroupType=CORE,InstanceCount=2,InstanceType=m3.xlarge
```

#10. Check the cluster status for "Cluster ready to run steps."
```sh
aws emr describe-cluster --cluster-id j-3VK2ZZEGD4P9O --query Cluster.Status.StateChangeReason.Message
```


spark-submit s3://emr-handson-rsp/py-example/Chopaliser.py s3://emr-handson-rsp/input/chopratings.csv s3://emr-handson-rsp/py-output
spark-submit --class com.amazonaws.emr.example.Chopaliser s3://emr-handson-rsp/java-example/chopalise-1.0-SNAPSHOT.jar s3://emr-handson-rsp/input/chopratings.csv s3://emr-handson-rsp/java-output




