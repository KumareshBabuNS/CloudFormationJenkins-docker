# Jenkins Cloudformation Template

## Overview:
This is a good example to learn many cool things around CloudFormation. Fun to play with. 
This repository contains a number of components to automatically deploy Jenkins in to a VPC in your own AWS account. Jenkins will run on a Docker Container on an Ubuntu host and will be self-healing. This is achieved by putting the instance in an autoscaling group with a minimum and maximum of 1. Jenkins will back up config, jobs, job history to an S3 bucket in your own account. On failure the autoscaling group will terminate the instance and launch a new one restoring from your latest backup. You should take the approach of a S3 lifecycle policy that prunes the backups every x days. Doing this lets me restore to a 1 hour RPO at any point over a x days period.


## Things to do before and after stack creation:
* Specify an S3 Bucket (make sure to setup a Life Cycle of x days.
* Specify an SNS topic with email subscriptions for notifications about Jenkins system metrics (full disk etc).
* Specify the security group ID of a bastion host in your account to allow secure SSH access to the Jenkins instance.
* Specify the VPC you wish to launch the instance in.
* Change the AMI's to ones created yourself using `/packer/packer.json` if you wish to use the Jenkins instance to pull private images.

## Things created by this CloudFormation stack
* CloudFormation waitHandle - To give time for the docker download of the jenkins image
* AutScaling LaunchConfiguration - To tell what AMI to lunch and what user data to install on the server.
* AutScaling Group - For the self healing - at this time this is not setup for scaling the application up and down.
* Rout53 (DNS) - You need to have a zone already setup in  your AWS account, this will add <application name>.<your Zone already in your account>
* Role/Policy - to Enable the Jenkins server to talk with S3/CloudWatch/CloudWatch Log/CloudFormation
* Security Groups
** ELB Security Group, allowing only 80 and 443
** Jenkins Security Group, allowing only 8080 to ELB  and 22, 8080 to Bastion security Group
* Elastic Load Balancer, port 80 mapping to Jenkins port 8080
* CloudWatch Alarms - sent to SNS topic
** Memory Utilization (5 min)
** CPU Utilization (5 min)
** Disk Utilization (5 min)
* CloudWatch Log Streams
** system logs - containing system information of the Ubuntu host running the container(s).
** container logs - to view any issues with containers running on your host
** cloud init output - so you can view the output of the AutoScaling group userdata
** docker logs - Docker Container Logs sending std out of your Jenkins container to the stream


## Notes
* Go to CloudWatch logs and open the log stream created by the stack, navigate to the container-logs stream and search the stream for the jenkins initial password.
* TODO: Add support for SSL certificate. (Cloud formation will need to be modify to add the SSL cert to the ELB on creation)
* TODO: The Policy should be updated with more specific resources and less access over those AWS services
* TODO: CloudWatch data are not getting push, need to look into why we are getting error: CloudWatch-PutInstanceData: Failed to call EC2 to obtain Auto Scaling group name.

## Docker components:
I have created a packer install script, you can install Packer on your own machine following the instructions at: https://www.packer.io/downloads.html.

The packer image will create an AMI in your account, and copy it to the regions specified in your file. I have chosen `us-east-1` for my template. The template in `cloudformation/jenkins.json` will need to be updated to use the AMI's in your account if you want. The one I'm using is the Amazon Ubuntu 14 via the region mappings section.

The reason you want to use your own account is I have included a section a file in `packer/config.json` that you can update with your Docker registry token to allow your Jenkins machine to pull private images.

Additionally, the Docker container is configured to use the DOOD (docker outside of docker) configuration. This will allow your Jenkins server to build docker images and run docker-compose unit tests as part of jenkins jobs. More information on the DOOD configuration can be found at: https://github.com/axltxl/docker-jenkins-dood this contains the reference material used to get this up and running.

When you are ready to build the AMI's in your own account, adjust the `packer/config.json` file and execute the command: `packer build packer.json`.

The Dockerfile in this repository is built in a public repository in quay.io.

## AWS components:
The Jenkins template comes with a number of components that probably require a bit of further explanation, they are as follows:
* TODO: Need to check the access, CloudWatch custom metrics which are emailed to the addresses in the SNS topic specified in the parameters section. including:
  * EC2 Instance CPU usage.
  * EC2 Instance Memory usage.
  * EC2 Instance disk space usage.
* IAM Role providing the Jenkins instance with **SOME ADMIN ACCESS** to the account it is launched in.
* Route 53 record creation is created by you specifying a Route53 zone in your AWS Account, this is updated at the end of the stack creation so you can log in to the Jenkins instance using this.

## Credit:
This template has used a lot of the concepts from the repository at https://github.com/thefactory/cloudformation-jenkins and https://github.com/TheRealDwright/jenkins I'm unsure if those repository are still being maintained.


## How to Run
* Set your CLI access to be an admin
* aws cloudformation create-stack \
     --template-body file://jenkins-all.json \
     --stack-name pipelinestack \
     --capabilities CAPABILITY_IAM \
     --parameters \
         ParameterKey=keyName,ParameterValue=thibeaultKeyPair \
         ParameterKey=dockerImage,ParameterValue='quay.io/therealdwright/jenkins:latest' \
         ParameterKey=instanceType,ParameterValue=t2.micro \
         ParameterKey=dnsPrefix,ParameterValue=jenkins \
         ParameterKey=dnsZone,ParameterValue=heffermangulch.com \
         ParameterKey=myPublicIp,ParameterValue='0.0.0.0/0' \
         ParameterKey=s3Bucket,ParameterValue=pipeline-backup9887 \
         ParameterKey=publicSubnets,ParameterValue='subnet-fa98afa3\,subnet-26adb051\,subnet-cbaa62f6\,subnet-eb1135c0' \
         ParameterKey=privateSubnets,ParameterValue='subnet-fa98afa3\,subnet-26adb051\,subnet-cbaa62f6\,subnet-eb1135c0' \
         ParameterKey=vpcId,ParameterValue=vpc-394f9c5d \
         ParameterKey=bastionHostSecurityGroup,ParameterValue=sg-80b256fa \
         ParameterKey=snsTopic,ParameterValue='arn:aws:sns:us-east-1:724662056237:myEmail'