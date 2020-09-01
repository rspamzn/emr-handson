# EMR Hands-on Workshop

## Section 1
Objective : To understand the configuration options in creating a EMR cluster using AWS CLI. We will start creating an EMR cluster from the scratch and add 2 steps in the end.
![alt text](https://docs.aws.amazon.com/emr/latest/ManagementGuide/images/vpc_with_private_subnet_v3a.png)


### Preparation :
The dependencies needed for this workshop are packaged here - https://bit.ly/3hOv3eO
Download this to a temporary location.
Upload the input/chopratings.csv file into your S3 Bucket (or create a new bucket)
Upload the java-example/chopalise-1.0-SNAPSHOT.jar into a Bucket


### Execution Steps :
1. Our VPC will be a very small range /24, with our subnet space using a /28 netmask. After executing the create-vpc command below, please copy the vpc-id from the response
```sh
aws ec2 create-vpc --cidr-block 10.20.30.0/24 --instance-tenancy default
aws ec2 create-tags --resources <VPC-ID> --tags Key=Name,Value=vpc-emr-handson
```

2. Let’s create a subnet to go along with the VPC, and specify the VPC ID and the range for the subnet. In this example, we will use 10.20.30.0/28.
```sh
aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.20.30.0/28
aws ec2 create-tags --resources <SUBNET-ID> --tags Key=Name,Value=subnet-emr-handson
```

3. We need a route table with a public Internet gateway. We will issue the create-route-table command with the VPC ID from earlier. Next, we will create the default route with the create-route command.
```sh
aws ec2 create-route-table --vpc-id <VPC-ID>
aws ec2 create-tags --resources <ROUTE-TABLE-ID> --tags Key=Name,Value=rtb-emr-handson
```

4. Because it is required by EMR, we will create a new internet gateway.
```sh
aws ec2 create-internet-gateway
aws ec2 create-tags --resources <INTERNET-GATEWAY-ID> --tags Key=Name,Value=igw-emr-handson
```

5. Next, we attach the Internet gateway to the VPC:
```
aws ec2 attach-internet-gateway --internet-gateway-id <INTERNET-GATEWAY-ID> --vpc-id <VPC-ID>
```

6. We will make the Internet gateway the default route using the Internet gateway ID and route table ID from earlier.
```sh
aws ec2 create-route --route-table-id <ROUTE-TABLE-ID>  --destination-cidr-block 0.0.0.0/0 --gateway-id <INTERNET-GATEWAY-ID>
```

7. And attach the route table:
```sh
aws ec2 associate-route-table --route-table-id <ROUTE-TABLE-ID>  --subnet-id <SUBNET-ID>
```

8. We need to check to see if DNS hostnames are enabled. If they are not, we will enable them.
```sh
aws ec2 describe-vpc-attribute  --vpc-id <VPC-ID> --attribute  enableDnsHostnames
aws ec2 modify-vpc-attribute --vpc-id <VPC-ID> --enable-dns-hostnames
```

9. Create EMR Default roles
```sh
aws emr create-default-roles
```

10. Create a keypair
```sh
aws ec2 create-key-pair --key-name emr-handson-keypair --query 'KeyMaterial' --output text > ~/Downloads/MyKeyPair.pem
```
 
11. Finally, we have everything in place to launch a cluster inside of the VPC successfully. We can use the following command to launch a test cluster
```sh
aws emr create-cluster \
   --name emr-handson-cluster \
   --emrfs Consistent=true \
   --use-default-roles \
   --applications Name=Spark \
   --ec2-attributes KeyName=emr-handson-keypair,SubnetId=<SUBNET-ID> \
   --release-label emr-6.1.0 \
   --instance-groups InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m4.xlarge InstanceGroupType=CORE,InstanceCount=2,InstanceType=m4.xlarge
```

12. Check the cluster status for "Cluster ready to run steps." (This will show null for the initial few seconds)
```sh
aws emr describe-cluster --cluster-id <CLUSTER-ID> --query Cluster.Status.StateChangeReason.Message
```

13. Submit a step - Spark Java Application. Replace the S3 Paths in the Args parameter before submitting
```sh
aws emr add-steps --cluster-id <CLUSTER-ID> --steps Type=Spark,ActionOnFailure=CONTINUE,Args=--class,com.amazonaws.emr.example.Chopaliser,s3://emr-handson-rsp/java-example/chopalise-1.0-SNAPSHOT.jar,s3://emr-handson-rsp/input/chopratings.csv,s3://emr-handson-raja/java-output2
aws emr describe-step --cluster-id <CLUSTER-ID> --step-id <STEP-ID>
```

14. Submit a step - Spark Python Application. Replace the S3 Paths in the Args parameter before submitting
```sh
aws emr add-steps --cluster-id j-1G7IZMXZDF9M1 --steps Type=Spark,ActionOnFailure=CONTINUE,Args=s3://emr-handson-rsp/py-example/Chopaliser.py,s3://emr-handson-rsp/input/chopratings.csv,s3://emr-handson-raja/java-output4
aws emr describe-step --cluster-id <CLUSTER-ID> --step-id <STEP-ID>
```
