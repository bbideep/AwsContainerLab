
# Building Services using .NET Core, Cross-Platform Tools, and AWS Fargate
Amazon Web Service (AWS) Fargate is a compute cluster engine that automatically manages containerized application deployment configuration.  There is no need to provision, manage, or scale any underlying Amazon EC2 compute infrastructure.

Fargate works with Amazon Elastic Container Service (ECS) and can run microservices developed in many programming languages or application frameworks including Java, .NET Core, Python, Node.js, Go, or Ruby on Rails. 

There are numerous methods to develop, build, and deploy .NET Core solutions using container hosting services such as ECS, Fargate, and EKS. 

The most accessible approach is to utilize the [AWS Toolkit for Visual Studio](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/deployment-ecs-aspnetcore-fargate.html) which provides nicely integrated project templates and publishing wizards. 
An extension for [Visual Studio Code](https://github.com/aws/aws-toolkit-vscode) also offers a nice cross-platform tooling experience.

Command-line tooling options include the [AWS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI.html), the [AWS ECS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI.html), 
or the [AWS .NET Core Global Tools](https://aws.amazon.com/blogs/developer/net-core-global-tools-for-aws/).  And, even combinations of these script libraries.

A tool comparison matrix is beyond the scope of this article. Instead, this article is intended for the command-line afficionados who may need to gain experience leading towards script-based automation of their build-to-publish pipeline. 
All of the steps taken herein would be automated in a real-world DevOps environment.

Herein, we illustrate building, deploying, and running a .NET Core containerized solution using cross-platform command line tools.

* Estimated Completion Time : 60 minutes
* Requirements : A laptop or workstation (Windows, Mac, Linux) with a Terminal application. Refer to  <a href="#appendix-a">**Appendix A**</a> herein.
* An [AWS Account](https://aws.amazon.com/account/)



### Reference Architecture

A reference architecture for an AWS Fargate deployment should specify artifacts such as a VPC, Subnets, Load Balancer, Internet Gateway, Elastic Network Interface (ENI), AWS Fargate Task, Network ACLs, and Security Groups. The architectural choices for VPC Networking, Load Balancing, and Container Networking will vary based upon solution requirements.

Herein, our example ASP.NET Core application will serve traffic from the Internet.  Hence, we will deploy containers in a public VPC Subnet using a Public Load Balancer option.

Refer to this [AWS Labs github project](https://github.com/awslabs/aws-cloudformation-templates/tree/master/aws/services/ECS) for solution architecture options illustrations.

Figure 1 illustrates a corresponding reference architecture. 

![Architecture](https://github.com/awslabs/aws-cloudformation-templates/blob/master/aws/services/ECS/images/public-task-public-loadbalancer.png)


![Picture1](./images/aspnetcorefargate.jpg)
**Figure 1 � Reference Architecture**

To implement this architecture, we will do the following:

* Containerize the ASP.NET core application.
* Configure the reverse-proxy server.
* Containerize the Nginx reverse-proxy server.
* Create the Docker Compose file.
* Push container images to Amazon ECR.
* Create the ECS cluster.
* Create an Application Load Balancer.
* Create an AWS Fargate Task definition.
* Create the Amazon ECS service.

<a id='dev-env'></a>
### Development environment
Your development environment needs to have the following minimal tools configuration.

**A Dev-Environment with the following development tools and services.** 
1. .NET Core  (https://dotnet.microsoft.com/download/dotnet-core)
2. AWS Command Line Tools (aws)  (https://aws.amazon.com/cli/)
3. AWS .NET Core Global Tools for ECS (dotnet-ecs)  (https://github.com/aws/aws-extensions-for-dotnet-cli)
4. Docker Tools and Services (docker) (https://www.docker.com/)
5. Docker Compose (docker-compose)  (https://docs.docker.com/compose/overview/)
6. Nginx (https://aws.amazon.com/amazon-linux-2/faqs/#Amazon_Linux_Extras)


**Your Lab Workstation - A Remote AWS Linux AMI Instance**
1. For this lab, we utilize an EC2 hosted **.NET Core 2.1 with Amazon Linux 2 - Version 1.0** instance.  This Amazon Machine Image (AMI) is preconfigured with both **AWS CLI** and the **dotnet** runtime.  It doesn't include **AWS .NET Core Global Tools** nor **docker**, but we will install those.
2. Don't know how to create an EC2 instance?  Please refer to step-wise documentation at (https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html). 
3. When creating your AMI instance, be sure to select the **.NET Core 2.1 with Amazon Linux 2 - Version 1.0** AMI-type.  We recommend utilizing a **t2.large** or better host-machine type. 
4. Please refer to a companion lab at (http://www.awslab.io/dotnet/launchingec2windows/) for general instructions on deploying an EC2 instance. But, instead of creating a Windows EC2 instance, choose the **.NET Core 2.1 with Amazon Linux 2 - Version 1.0** image-type. 
5. When creating your EC2 instance, be sure to either download (or select an existing) **PEM** key file for subsequent use for secure shell (SSH) logon from your local terminal application.
6. Refer to  <a href="#appendix-a">**Appendix A**</a> herein for instructions on accessing your AMI instance via SSH. 

**Proceed with installation of the following tools only after completion of the above workstation guidance.  All further lab instructions are to be executed from a terminal window remoting into your EC2 instance via SSH.**

**Update and configure your Lab Workstation**

1. Using your local Terminal application, SSH into the remote Lab Workstation.
``` shell
ssh -i MYKEYPAIR.pem ec2-user@myEc2IpAddress

logon examples:
ssh -i ~\downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
ssh -i %HOME%\Downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
```
2. Update the current image package configuration.
``` shell
sudo yum update -y
```
3. Install the AWS .NET Core Global Tools for ECS.
``` shell
dotnet tool install -g Amazon.ECS.Tools
```
4. Install Docker Tools and Services.
``` shell
sudo yum install -y docker
sudo service docker start
```
5. Add the **ec2-user** account to the **docker group** to enable use without root privileges (i.e. sudo).
``` shell
sudo usermod -a -G docker ec2-user
```
6. Install the Nginx service supporting our Reverse Proxy configuration.
``` shell
sudo amazon-linux-extras install nginx1.12
``` 
7. Install Docker Compose for testing our containers via localhost. Ensure that `docker-compose` has the execute attribute enabled.
``` shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
8. Logout and logon again in order to effect the `ec2-user` user group privilege change made previously.
``` shell
logout
```
``` shell
logon examples:
ssh -i ~\downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
ssh -i %HOME%\Downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
```
9. Test these installed packages.
``` shell
aws --version
```
``` shell
dotnet --version
```
``` shell
dotnet-ecs --version
```
``` shell
docker --version
docker ps
```
``` shell
docker-compose --version
```

### Create an ASP.NET Core MVC application
Let's leverage the `dotnet` Global Tools project templates for creating, building and publishing an ASP.NET Core MVC application. 

From your SSH terminal session, invoke the following command sequence.

``` shell
cd ~/    

mkdir myproject     

mkdir myproject/mywebapp

cd myproject/mywebapp

dotnet new -all  (note: You should see a listing of available .NET project templates.  We will utilize the ASP.NET Core Web App MVC template.)

dotnet new mvc

dotnet restore

dotnet build 

dotnet publish -c "release"
```

The above command set creates an ASP.NET Core MVC web application, restores dependencies, builds the application, and publishes the application package to the release folder.



### Containerize your ASP.NET Core application
create a file named `Dockerfile` within the root `mywebapp` folder and copy the following `docker` build script and create a file named `Dockerfile` within the root `mywebapp` folder.  (If using a Linux workstation for your development environment, consider using the `nano` text editor.)

``` shell 
cd ~/myproject/mywebapp
nano Dockerfile          
```
``` docker
# Copy this script into ~/myproject/mywebapp/Dockerfile

FROM microsoft/dotnet:2.1-aspnetcore-runtime AS base

FROM microsoft/dotnet:2.1-sdk AS build

WORKDIR /mywebapp
COPY bin/release/netcoreapp2.1/publish .

ENV ASPNETCORE_URLS http://+:5000
EXPOSE 5000

ENTRYPOINT ["dotnet", "mywebapp.dll"]
```

**Note:** Alternatively, use the following command to download a copy of this script file.
``` shell
curl -L -o Dockerfile https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/mywebapp/Dockerfile
```

**Note**: This lab is written using .NET Core Runtime version 2.1, which is the runtime version currently deployed within AWS AMI images.  This will change over time.  Use the `dotnet --version` command to identify the currently installed runtime and make corresponding changes within your `Dockerfile` as needed.

The above `Dockerfile` definition creates an ASP.NET Core container and copies the application deployment package from the `bin/release/netcoreapp2.1/publish` folder into a mywebapp folder within the container.
It also leverages Kestrel as the web server with a default port of 5000 for the web application.  

By default, ASP.NET Core uses Kestrel as the local web server.  Kestrel is a lightweight HTTP server and is typically used for local development purposes. Several other full-featured web-server hosting options are available.  See platform dependent ASP.NET Core web-server hosting options [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/?view=aspnetcore-2.2&tabs=windows).
 
For capabilities such as serving static content, caching requests, compressing responses, and terminating SSL, you can optionally leverage a dedicated reverse-proxy server such as NGINX.
   


### Create an NGINX reverse proxy container

Nginx can act as both the HTTP and reverse-proxy server. Nginx is highly adopted because of its asynchronous, event-driven architecture that allows it to serve thousands of concurrent requests with a low-memory footprint.

In this solution, deploy a Nginx `reverseproxy` container in front of the application `mywebapp` container, to be defined within an AWS Fargate Task.

The reverse-proxy configuration file `nginx.conf` should be specified as follows:

Navigate to the `myproject` directory and create a sub-directory named `reverseproxy`. 

``` shell
cd ~/myproject  
mkdir reverseproxy
cd reverseproxy
```

Create a new text file named `nginx.conf`.

``` shell 
nano nginx.conf
```

Edit the `nginx.conf` file, add the following definition. 

``` docker
# Copy this script into file ~/myproject/reverseproxy/nginx.conf
 
worker_processes 4;
 
events { worker_connections 1024; }
 
http {
    sendfile on;
 
    upstream app_servers {
        server mywebapp:5000;
    }
 
    server {
        listen 80;
 
        location / {
            proxy_pass         http://app_servers;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
        }
    }
}
```

**Note:** Alternatively, use the following command to download a copy of this script file.
``` shell
curl -L -o nginx.conf https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/reverseproxy/nginx.conf
```

As illustrated above, we specify service name `mywebapp`, listening on port 5000, within the `upstream app_servers` section. The Nginx `reverseproxy` server listens on port 80. 

However, when our application is hosted within an AWS Fargate Task, we need to change the value of `upstream app_servers` to `127.0.0.1:5000`. 

When the Fargate Task runs in the default `awsvpc` networking mode it will use the local loopback interface of `127.0.0.1` to connect to the other container services defined as a part of the overall Fargate Task.
More on this in later sections.

Within the `reverseproxy` folder, create a `Dockerfile` for containerizing the Nginx reverse proxy.  Add the following Doocker script commands.

``` shell
cd ~/myproject/reverseproxy 
nano Dockerfile
```

``` Docker
# Copy this script into file ~/myproject/reverseproxy/Dockerfile

FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```

**Note:** Alternatively, use the following command to download a copy of this script file.
``` shell
curl -L -o Dockerfile https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/reverseproxy/Dockerfile
```

The `Dockerfile` script creates a container and copies the `nginx.conf` file in the `reverseproxy` folder to the `/etc/nginx/` folder inside our container.

### Test locally using Docker Compose

Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application�s services. Then, with a single command, you create and start all the services from your configuration.

Using Compose is basically a three-step process:

* Define your app�s environment with a `Dockerfile` so it can be reproduced anywhere.

* Define the services that make up your app in `docker-compose.yml` so they can be run together in an isolated environment.

* Run `docker-compose up` and Compose starts and runs your entire app.

If your lab development environment is based upon a production Windows Server or a Linux image, then Compose may not be installed by default. Recall that we have already install `docker-compose` earlier in this lab. 

Refer to the Docker Compose overview page (https://docs.docker.com/compose/overview/). 


### Create the Docker Compose file
Now let's compose both of our `mywebapp` and `reverseproxy` containers as Docker application by defining the `docker-compose.yml` as follows.

``` shell
cd ~/myproject 
nano docker-compose.yml
```
``` yaml
# Copy this script into file ~/myproject/docker-compose.yml

version: '2'
services:
  mywebapp:
    build:
      context: ./mywebapp
      dockerfile: Dockerfile
    expose:
      - "5000"
  reverseproxy:
    build:
      context: ./reverseproxy
      dockerfile: Dockerfile
    ports:
      - "80:80"
    links :
      - mywebapp
```

**Note:** Alternatively, use the following command to download a copy of this script file.
``` shell
curl -L -o docker-compose.yml https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/docker-compose.yml
```

The `docker-compose.yml` file defines two services. The first service, `mywebapp`, exposes port 5000 and depends upon the service, `reverseproxy`, exposed via port 80. 

The `reverseproxy` service runs the Nginx container on port 80 and also exposes port 80 to outside world. It also links with the service `mymvcweb`. 

The links specification works for Docker Service Composition within our development environment. However, when you convert this into an ECS service definition for Fargate Tasks, these links are not supported via the `awsvpc` networking mode. More detail follows herein.

Now build and run these containers as a cohesive service within the local development environment by issuing the following commands.

``` shell
cd ~/myproject 
docker-compose build
```

The `docker-compose build` should produce results similar to the following (container ids will vary based on your environment).

```
Building mywebapp
Step 1/6 : FROM microsoft/aspnetcore:2.1
 ---> c8e388523897
Step 2/6 : WORKDIR /mywebapp
 ---> Using cache
 ---> a84539366440
Step 3/6 : COPY bin/Release/netcoreapp2.1/publish .
 ---> Using cache
 ---> 20413b534ce2
Step 4/6 : ENV ASPNETCORE_URLS http://+:5000
 ---> Using cache
 ---> 12aa8b85ecf8
Step 5/6 : EXPOSE 5000
 ---> Using cache
 ---> a1045008f67d
Step 6/6 : ENTRYPOINT ["dotnet", "mywebapp.dll"]
 ---> Using cache
 ---> d63c60da7a9d
Successfully built d63c60da7a9d
Successfully tagged myproject_mywebapp:latest
Building reverseproxy
Step 1/2 : FROM nginx
 ---> 73acd1f0cfad
Step 2/2 : COPY nginx.conf /etc/nginx/nginx.conf
 ---> Using cache
 ---> 4c5e493bc01d
Successfully built 4c5e493bc01d
Successfully tagged myproject_reverseproxy:latest
```

Now, we need to both run our containerized web application and concurrently browse to the application.
An easy localhost testing approach is to simply open a second logon session, run our application, and invoke it from our 
first logon session.

From your desktop, open a new Terminal window and again SSH into the remote Lab Workstation (keeping your first SSH session open).
``` shell
ssh -i MYKEYPAIR.pem ec2-user@myEc2IpAddress

logon examples:
ssh -i ~\downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
ssh -i %HOME%\Downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
```

Next, within your new SSH Terminal window, invoke the following command to start our linked container services.

``` shell 
cd ~/myproject
docker-compose up
```

Observe results similar to the following.  Our `mywebapp` is now running and receiving HTTP requests via localhost port 5000. 
Our `reverseproxy` is also running but listening for HTTP requests on localhost port 80.  

```
Creating myproject_mymvcweb_1     ... done
Creating myproject_mymvcweb_1     ... 
Creating myproject_reverseproxy_1 ... done
Attaching to myproject_mymvcweb_1, myproject_reverseproxy_1
mymvcweb_1      | warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
mymvcweb_1      |       No XML encryptor configured. Key {2059191a-ff77-4fa9-a968-0962d7a8f10b} may be persisted to storage in unencrypted form.
mymvcweb_1      | Hosting environment: Production
mymvcweb_1      | Content root path: /mywebapp
mymvcweb_1      | Now listening on: http://[::]:5000
mymvcweb_1      | Application started. Press Ctrl+C to shut down
```

Leave this SSH session running, and return to your original SSH Terminal window.  Execute the following commands to send an HTTP request to our `mywebapp` application.  Note the response is saved to a `mytest.out` file.

``` shell
cd ~/myproject
curl -o mytest.out http://localhost:80
```

Next, view the `mytest.out` file to confirm that the default view of our ASP.NET Core MVC application's `index.cshtml` page was returned.

``` shell
more mytest.out
```
This confirms that our `reverseproxy` is indeed forwarding requests to our `mywebapp` and also proxying responses.
 
After completing the testing in the local development environment, return to your second SSH session and stop the container hosting services.

``` shell
From your second SSH terminal window,
press 'ctrl+c' to stop your web services
```

Return to your first Terminal window, issue the following commands to clean up with `docker-compose`.

``` shell
docker-compose ps
docker-compose stop
docker-compose rm
docker-compose images
docker-compose rmi 'containerimageid'
```


### Push container images to the Amazon Elastic Container Registry (ECR) using Docker

Next, push the container images from the local environment to Amazon ECR so that the container images are available before creation of your AWS Fargate cluster.

Before you deploy this application to ECR, first modify the `upstream app_servers` attribute in the `nginx.conf` file must be set with the value of `127.0.0.1:5000`. This enables communication with the upstream application container listening on port 5000 within an ECS deployment.

``` shell
cd ~/myproject/reverseproxy 
nano nginx.conf
```

Edit the `nginx.conf` file, change the `upstream app_servers` attribute to `127.0.0.1:5000` as illustrated below. 

``` docker
worker_processes 4;
 
events { worker_connections 1024; }
 
http {
    sendfile on;
 
    upstream app_servers {
        server 127.0.0.1:5000;
    }
 
    server {
        listen 80;
 
        location / {
            proxy_pass         http://app_servers;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
        }
    }
}
```
``` shell
press 'ctrl+s' to save the file 
press 'ctrl+x' to exit the nano editor
```

Execute a build to update your local container images.

``` shell
cd ~\myproject
docker-compose build
```

View your Docker images.
``` shell
Docker images
```

Prior to invoking AWS Services, ensure that your **AWS CLI** tooling is configured with your lab account credentials.  

Modify the prompting command line as indicated and also specify your default [region](https://docs.aws.amazon.com/general/latest/gr/rande.html).  
Specifying the region here is important because subsequent commands leverage the `--region` attribute implicitly via your configured **AWS CLI** profile.

``` shell
cd ~/myproject

aws configure
  AWS Access Key ID [None]: <PASTE YOUR ACCESS KEY ID HERE>
  AWS Secret Access Key [None]: <PAST YOUR ACCESS KEY HERE>
  Default region name [None]: us-west-2  <CHOOSE YOUR PREFERRED REGION>
  Default output format [None]: json
```

Create two ECR [repositories](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html), namely `mywebapp` and `reverseproxy`.  Use `get-logon` to request key-based secure credentials for the `Docker` service to use in order to push our containers to ECR.

``` shell
aws ecr create-repository --repository-name mywebapp > mywebapp-repo.txt
aws ecr get-login --no-include-email > mywebapp-login.sh
```
``` shell
aws ecr create-repository --repository-name reverseproxy > myreverseproxy-repo.txt
aws ecr get-login --no-include-email > myreverseproxy-login.sh
```

**Note:** The output from each command above is saved to corresponding text files as illustrated. Use `cat` to view file contents.  Make a note of the repository URL at the end of the text.

Next, use the generated `Docker login` command files to enable `docker` to access to each repository. 

``` shell
eval < mywebapp-login.sh
```
``` shell
eval < myreverseproxy-login.sh
```

Note that ECR login keys are valid for only 12 hours.  Afterwards, another logon request must be issued.
```
eval "$(aws ecr get-login --no-include-email)"
```

Use the following command to retrieve information about your ECR repositories and corresponding account URLs.

``` shell
aws ecr describe-repositories
```

View your Docker images.
``` shell
Docker images
```

For the remote ECR repository (i.e. `mywebapp`), create a corresponding tag. Carefully, modify this command per your specific account number and region.

``` shell
docker tag myproject_mywebapp:latest your-aws-account-number.dkr.ecr.your-region.amazonaws.com/mywebapp:latest 
```

For the remote ECR repository (i.e. `reverseproxy`), create a corresponding tag. Carefully, modify this command per your specific account number and region. 
Note that the repository name (local and remote) is appended with the `:tag-label`.

``` shell
docker tag myproject_reverseproxy:latest your-aws-account-number.dkr.ecr.your-region.amazonaws.com/reverseproxy:latest 
```

Push the `mywebapp` image to the remote ECR `mywebapp` repository.  Note that the ECR "URL" does not include the previously illustrated  `:tag-label`.

``` shell
docker push yourawsaccountnumber.dkr.ecr.your-region.amazonaws.com/mywebapp 
```

Push the `reverseproxy` image to the remote ECR `reverseproxy` repository.

``` shell
docker push yourawsaccountnumber.dkr.ecr.your-region.amazonaws.com/reverseproxy 
```


## Create an ECS Cluster with a Fargate Task using CloudFormation


Let's review our target solution architecture.

![Picture2](./images/aspnetcorefargate.jpg)

It's a common misconception that Fargate clusters require no infrastructure to be configured.  In fact, our VPC architecture includes quite a bit of Infrastructure as Code specification.

Actually, Fargate works with ECS to relieve you only of EC2 instance maintenance.  We still need to establish artifacts of our VPC that define our clustering environment.

When creating AWS Infrastructure components, it's convenient to utilize a CloudFormation template.  There are many advantages to this approach including an ability to version control our clustering environment.  

A CloudFormation template is used to create a **stack** representing our architecture.  Use the following command to download a copy our CloudFormation template.

``` shell
cd ~/myproject
curl -L -o my-public-vpc.json https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/my-public-vpc.json
```

Using `more`, `cat`, or `nano` to review this template file. In all probability it will require no changes. Note the specification of all VPC, Subnet, Application Load Balancer, ENI and other architectural requirements. 
Our Fargate Task elements remain to be defined.

First, validate the template file.
```
aws cloudformation validate-template --template-body file://my-public-vpc.json
```

Next, create your CloudFormation stack.  Notice the returned `StackId`. Also note the introduction of the  `--capabilities CAPABILITY_IAM` parameter.  This represents your acknowledgement that the script will modify IAM security groups within your AWS account.
`
aws cloudformation create-stack --stack-name my-public-vpc-stack --capabilities CAPABILITY_IAM --template-body file://my-public-vpc.json 
`

Check the status of our stack creation.  It may take a few minutes to spin-up this infrastructure.
```
aws cloudformation describe-stack-events --stack-name my-public-vpc-stack
```

Confirm completion of the stack creation.
```
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE
```



Create an ECS Fargate Task

```
curl -L -o my-fargate-service.yml https://raw.githubusercontent.com/awslabs/aws-cloudformation-templates/master/aws/services/ECS/FargateLaunchType/services/public-service.yml

```




DELETE BELOW
## Create a Cluster with a Fargate Task using the AWS CLI

Let's create a Fargate cluster using the [AWS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html) tooling that we previously installed.

Create your own uniquely named cluster with the following command.

``` shell
aws ecs create-cluster --cluster-name myfargatecluster
```

Register a Task definition as follows.  Task definitions specify how to create and configure your container hosting environment.


DELETE ABOVE


## Create a Cluster with a Fargate Task using the Amazon ECS CLI

Let's create a Fargate cluster using the [Amazon ECS CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-fargate.html) tooling.

### Install the ECS CLI

``` shell
sudo curl -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest
```
``` shell
sudo chmod +x /usr/local/bin/ecs-cli
```
``` shell
ecs-cli --version
```

### Create a Task execution IAM role

Create a `task-execution-assume-role.json` with the following contents. Amazon ECS needs permissions so that your Fargate task can store logs in CloudWatch. These permissions are covered by the task execution IAM role.

``` shell
cd ~/myproject 
nano task-execution-assume-role.json
```
``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Note:** Alternatively, use the following command to download a copy of this script file.
``` shell
curl -L -o task-execution-assume-role.json https://raw.githubusercontent.com/UsefulEngines/staticfiles/master/awslab.io/Containers/Fargate/scripts/myproject/task-execution-assume-role.json
```

Create the task execution role within your default region.
``` shell
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json
```

Attach the task execution role policy within your default region.
```
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### Configure the Amazon ECS CLI

Create a cluster configuration, which defines the AWS region to use, resource creation prefixes, and the cluster name to use with the Amazon ECS CLI.
```
ecs-cli configure --cluster myfargatecluster --region your-region --default-launch-type FARGATE --config-name myfargatecluster
```

Create a CLI profile using your access key and secret key in order for the ECS CLI to accomplish API calls on your behalf.
```
ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name tutorial
```

### Create a Cluster and Security Group

Create an Amazon ECS cluster with the `ecs-cli up` command. Because you specified Fargate as your default launch type in the cluster configuration, this command creates an empty cluster and a VPC configured with two public subnets.

```
ecs-cli up
```

**Note:** This command may take a few minutes to complete as your resources are created. Capture the VPC and Subnet ID's that are created as they are used later.

Create a security group using the VPC ID from the previous output.

```
aws ec2 create-security-group --group-name "my-sg" --description "MySecurityGroup" --vpc-id "VPC_ID"
```

Add a security group rule to allow inbound access on port 80.
```
aws ec2 authorize-security-group-ingress --group-id "security_group_id" --protocol tcp --port 80 --cidr 0.0.0.0/0
```



This completes our illustration about how to host an ASP.NET Core MVC application and Nginx reverse proxy using AWS Fargate.


<a id='appendix-a'></a>
# Appendix A : Connect to your AWS Linux AMI Development Environment

### <i class="fab fa-windows" aria-hidden="true"></i> Windows Users: Using SSH to Connect

<i class="fas fa-comment" aria-hidden="true"></i> These instructions are for Windows users only.  There are several options for client terminal (i.e. shell) access to an AWS Linux AMI instance. This guide recommends using the Open Source application named **Cmder**.

If you are using a Mac or Linux lab computer, <a href="#ssh-MACLinux">skip to the next section</a>.

1. From your Windows workstation, install the useful **Cmder** command shell (https://cmder.net) enabling a more cross-platform terminal experience.  Be sure to select the "Download Full" option which includes 'git-for-windows' support and 'Unix' style commands. 
2. We recommend that you extract the Cmder.zip file to c:\Cmder and add that folder to your system PATH environment variable. 
3. Launch a Cmder session by clicking the C:\Cmder\Cmder.exe file from Windows Explorer.  This is your terminal window.
4. You will use SSH from a **Cmder** window to logon to your Amazon EC2 instance.

<a id='ssh-MACLinux'></a>
### Windows,<i class="fab fa-windows" aria-hidden="true"></i> Mac, <i class="fab fa-apple" aria-hidden="true"></i> and Linux <i class="fab fa-linux" aria-hidden="true"></i> Users

1. On your local client machine, open a terminal window.
2. Using your AWS Account, logon to the AWS Management Console.
3. Select the **EC2** service.
4. From the EC2 console, see your previously created **Amazon ECS-optimized Amazon Linux 2 AMI** instance within the list of hosted AMI instances.
5. Select your machine image and right-click to open a pop-up menu. Select the **Connect** option.
6. Note the displayed EC2 instance IP Address, URL, and SSH command line example. Copy this SSH command line to a text editor and correctly specify your **PEM** key-pair file name.
7. Execute the following commands from your terminal window.  Your key-pair file-name and Ec2IpAddress will differ from the example.

```plain
chmod 400 MYKEYPAIR.pem
ssh -i MYKEYPAIR.pem ec2-user@myEc2IpAddress

example:
chmod 400 ~\downloads\mykeypair.pem
ssh -i ~\downloads\mykeypair.pem ec2-user@ec2-34-219-242-224.us-west-2.compute.amazonaws.com
```
8. Type `yes` when prompted to allow a first connection to this remote SSH server.

Because you are using a key pair for authentication, you will not be prompted for a password.

<a href="#dev-env">Return to the top of this file</a>

<a id='ssh-after'></a>










