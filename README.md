# EMR Hands-on Workshop

## Section 1
Objective : To understand the configuration options in creating a EMR cluster using AWS CLI. We will start creating an EMR cluster from the scratch and add 2 steps to be executed in the end.
![alt text](https://docs.aws.amazon.com/emr/latest/ManagementGuide/images/vpc_default_v3a.png)


### Preparation :
The dependencies needed for this workshop are packaged here - https://bit.ly/3hOv3eO
Download this to a temporary location by clicking on the link. Unzip the downloaded file. There will be a parent directry(EMR-Handson-Dependencies) with 3 subdirectories(input, java-example, py-example)
Create a bucket to hold these files and upload the unzipped content using the steps below. These files will be used in the last few steps of this section.
```sh
aws s3api create-bucket --bucket emr-handson-<your initials here>
cd EMR-Handson-Dependencies
aws s3 sync . s3://emr-handson-<your initials here>
```

### Execution Steps :
>1. Our VPC will be a very small range /24, with our subnet space using a /28 netmask. After executing the create-vpc command below, please copy the vpc-id from the response
```sh
aws ec2 create-vpc --cidr-block 10.20.30.0/24 --instance-tenancy default
aws ec2 create-tags --resources <VPC-ID> --tags Key=Name,Value=vpc-emr-handson
```

>2. Let’s create a subnet to go along with the VPC, and specify the VPC ID and the range for the subnet. In this example, we will use 10.20.30.0/28.
```sh
aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.20.30.0/28
aws ec2 create-tags --resources <SUBNET-ID> --tags Key=Name,Value=subnet-emr-handson
```

>3. We need a route table with a public Internet gateway. We will issue the create-route-table command with the VPC ID from earlier. Next, we will create the default route with the create-route command.
```sh
aws ec2 create-route-table --vpc-id <VPC-ID>
aws ec2 create-tags --resources <ROUTE-TABLE-ID> --tags Key=Name,Value=rtb-emr-handson
```

>4. Because it is required by EMR, we will create a new internet gateway.
```sh
aws ec2 create-internet-gateway
aws ec2 create-tags --resources <INTERNET-GATEWAY-ID> --tags Key=Name,Value=igw-emr-handson
```

>5. Next, we attach the Internet gateway to the VPC:
```
aws ec2 attach-internet-gateway --internet-gateway-id <INTERNET-GATEWAY-ID> --vpc-id <VPC-ID>
```

>6. We will make the Internet gateway the default route using the Internet gateway ID and route table ID from earlier.
```sh
aws ec2 create-route --route-table-id <ROUTE-TABLE-ID>  --destination-cidr-block 0.0.0.0/0 --gateway-id <INTERNET-GATEWAY-ID>
```

>7. And attach the route table:
```sh
aws ec2 associate-route-table --route-table-id <ROUTE-TABLE-ID>  --subnet-id <SUBNET-ID>
```

>8. We need to check to see if DNS hostnames are enabled. If they are not, we will enable them.
```sh
aws ec2 describe-vpc-attribute  --vpc-id <VPC-ID> --attribute  enableDnsHostnames
aws ec2 modify-vpc-attribute --vpc-id <VPC-ID> --enable-dns-hostnames
```

>9. Create EMR Default roles
```sh
aws emr create-default-roles
```

>10. Create a keypair
```sh
aws ec2 create-key-pair --key-name emr-handson-keypair --query 'KeyMaterial' --output text > ~/Downloads/MyKeyPair.pem
```
 
>11. Finally, we have everything in place to launch a cluster inside of the VPC successfully. We can use the following command to launch a test cluster. Change the bucket if for logs and the subnet id
```sh
aws emr create-cluster \
   --name emr-handson-cluster \
   --log-uri s3://<BUCKET-ID-FOR-LOGS> \
   --emrfs Consistent=true \
   --use-default-roles \
   --applications Name=Spark \
   --ec2-attributes KeyName=emr-handson-keypair,SubnetId=<SUBNET-ID> \
   --release-label emr-6.1.0 \
   --instance-groups InstanceGroupType=MASTER,InstanceCount=1,InstanceType=m4.xlarge InstanceGroupType=CORE,InstanceCount=2,InstanceType=m4.xlarge
```

>12. Check the cluster status for "Cluster ready to run steps." (This will show null for the initial few seconds)
```sh
aws emr describe-cluster --cluster-id <CLUSTER-ID> --query Cluster.Status.StateChangeReason.Message
```

>13. Submit a step - Spark Java Application. Replace the S3 Paths in the Args parameter before submitting
```sh
aws emr add-steps --cluster-id <CLUSTER-ID> --steps Type=Spark,ActionOnFailure=CONTINUE,Args=--class,com.amazonaws.emr.example.Chopaliser,s3://emr-handson-rsp/java-example/chopalise-1.0-SNAPSHOT.jar,s3://emr-handson-rsp/input/chopratings.csv,s3://emr-handson-raja/java-output2
aws emr describe-step --cluster-id <CLUSTER-ID> --step-id <STEP-ID>
```

>14. Submit a step - Spark Python Application. Replace the S3 Paths in the Args parameter before submitting
```sh
aws emr add-steps --cluster-id j-1G7IZMXZDF9M1 --steps Type=Spark,ActionOnFailure=CONTINUE,Args=s3://emr-handson-rsp/py-example/Chopaliser.py,s3://emr-handson-rsp/input/chopratings.csv,s3://emr-handson-raja/java-output4
aws emr describe-step --cluster-id <CLUSTER-ID> --step-id <STEP-ID>
```

## Section 2
Objective : To use EMR notebooks to execute a pyspark script. 

>1. Create a notebook via the AWS console. Navigate to the EMR Service Page and select Notebooks option from the left menu bar. Select the 'Create Notebook' button.
>    * Provide the Notebook Name.
>    * Select 'Choose an existing cluster' radio button and choose the current active cluster.
>    * Leave the rest of the options to default.
>    * Click on the 'Create Notebook" button at the bottom and wait for the status of the notebook to change to Ready.

>2. Click on the 'Open in Jupyter' button.

>3. Once the notebook is opened, change the kernel to PySpark by selecting the 'Kernel->Change Kernel' menu on top.

>4. Open the notebook codes in (https://github.com/rspamzn/emr-handson/blob/master/notebook/emr-handson-notebook.ipynb). Copy each cell content into the Jupyter notebook and execute by clicking on the Run button on top. Remember to chagne the bucket-ids in the core wherever it appears.

