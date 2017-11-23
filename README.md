
[TOC]



# 1.Objective
The aim of this article is to provide guidance to migrate an ASP.NET MVC 4.6 or older application in to AWS ECS using containers. This will also cover the step-by-step instructions, cloud formation template and ECS task definition

# 2.	Why ASP.NET MVC 4.6 and Windows Containers?
The ASP.NET MVC 4.6 and older versions of ASP.net occupy a significant footprint in the enterprise web application space. As enterprises  move towards microservices for new or existing applications, containers are one of the stepping stones for migrating from monolithic to microservices architectures. Additionally, support for Windows Containers in Windows 10, Windows Server 2016, and Visual Studio Tooling support for Docker, simplifies the containerization of ASP.NET MVC apps.

# 3.	Pre-requisites
Ensure your development environment has the following setup as per this [Microsoft article](https://docs.microsoft.com/en-us/aspnet/mvc/overview/deployment/docker-aspnetmvc):

a) Visual Studio 2017 with latest updates
b) Docker for windows – version stable 1.13.0 or 1.12 Beta 26 (or newer versions)
c) Windows 10 Anniversary Update (or higher) or Windows Server 2016 (or higher)

If the web application was developed in earlier version of Visual Studio, it needs to be opened in Visual Studio 2017 which migrates all the settings to IDE.


The following is the sample ASP.NET MVC 4.6 application that we’ll be migrating.
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic1.jpg)

When the application is run in the development environment, it renders the following output.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic2.jpg)

# 4.	Containerization of ASP.NET MVC 4.6 application
These are the steps we will follow to containerize the ASP.NET MVC 4.6 application:
- Creation of Docker file
- Building a Docker image that will run ASP.NET MVC web app
- Run image locally
- Test in browser

## 4a.	Creation of Docker file
The Visual Studio 2017 support for Docker Tooling is leveraged to create Docker file.
Right click on Project and Add Docker Support

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic3.jpg)
 
This adds a Docker file to the current project and a Docker Compose project to the Solution.

The Docker file added by Visual Studio 2017 should look like this:

			
```
FROM microsoft/aspnet
ARG source
WORKDIR /inetpub/wwwroot
COPY ${source:-obj/Docker/publish} .

```

This file pulls the ASPNET Windows container image from the public DockerHub repository and pushes the binaries of the current project onto containers. This is enough for running locally in the development environment. To be able to run on Amazon ECS and for users to access the web service through Application Load Balancer (ALB), port 80 needs to be explicity exposed on the container. Update the Dockerfile to look like this:
```
FROM microsoft/aspnet
ARG source
WORKDIR /inetpub/wwwroot
COPY ${source:-obj/Docker/publish} .
EXPOSE 80

```
Right Click on the ASP.NET MVC Project -> Build

This completes compiling and building the ASP.NET MVC project.

## 4b.	Building a Docker image that will run ASP.NET MVC web app
The Docker Compose project in the Visual Studio Solution will be leveraged to build a Docker image that will run the ASP.NET MVC app. The Docker compose project added by Visual Studio 2017 should look like this.

 ![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic4.jpg)

The Docker compose.yml definition added by Visual Studio 2017 should look like this.
version: '2.1'


```
==services:
  awsecssample:
    image: awsecssample
    build:
      context: .\AWSECSSample
      dockerfile: Dockerfile==

```

The default docker-compose.yml will be leveraged for building the container image. 

Right click on docker-compose project -> Build.

A container image is built as per the Docker file definition. It can be verified by invoking ‘Docker Images’ command in terminal.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic5.jpg)

The step of building windows container image for ASP.NET 4.6 web application is complete.

##4c.	Starting container that runs the image 
The ‘docker run’ command is executed from the terminal to start the container that runs the image.


```
docker run -d --name aspnetcontainer awsecssample
```

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic6.jpg)
 
## 4d.	Test in browser

When the http://localhost is opened in the browser, it will not render the expected ASP.NET MVC app. With the current windows container release, we can’t browse to http://localhost.  This is a known behavior in WinNat. Please refer this article for more details

https://docs.microsoft.com/en-us/aspnet/mvc/overview/deployment/docker-aspnetmvc

The IP address of the container needs to be figured out by running this command.

```
docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" aspnetcontainer

``` 
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic7.jpg)

When the container IP is accessed, it renders the ASP.NET MVC 4.6 app running inside windows container.
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic8.jpg) 

This confirms that the ASP.NET MVC app is running successfully inside containers in the development environment

#  5.	Amazon EC2 Container Registry
The ASP.NET MVC which is running inside Windows Container (completed in previous section) needs to be scheduled and orchestrated in the Amazon EC2 container service. The Amazon EC2 container service can access container images stored in docker hub (private / public) or your organization’s container repository or AmazonEC2 Container registry. The focus of this article is to push the windows container image to the Amazon EC2 container registry. The Amazon EC2 container registry is a fully-managed Docker container registry that makes it easy for developers to store, manage and deploy Container images. It is deeply integrated with Amazon ECS, simplifying development to production workflow. 

## 5.1 Create repository
Each AWS root account gets an EC2 container registry (ECR) per region. Let’s create a ECR repository in a region of our choice.
 
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic9.jpg)
After successful creation of ECR Repository, the options of log in to Docker, tag image and push image will be available. 

## 5.2		  Log in to ECR 
The docker log in command needs to be retrieved to authenticate the Docker client into the registry.

aws ecr get-login --no-include-email --region ap-southeast-2

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic10.jpg)

Then, the Docker log in command needs to be invoked.
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic11.jpg) 

The log in is successful now.


## 5.3 Tag container image
The Container image that was built in the local development environment needs to be tagged with the ECR repository.

```
docker tag awsecssample:latest 065770805525.dkr.ecr.ap-southeast-2.amazonaws.com/awsecssample:latest

``` 
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic12.jpg)

## 5.4 Push container image
Run the docker push command to push the newly created image to the ECR repository.

```
docker push 065770805525.dkr.ecr.ap-southeast-2.amazonaws.com/awsecssample:latest
```
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic13.jpg)

 

The container image is stored in a compressed manner in the ECR repository. The actual size of this container image is around 11GB in the local development environment. In the ECR repository it’s size is around 465 MB, which is clearly evident about the compression of images. 

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic14.jpg)



The required permissions need to be provided to the container image so that ECS instances can start leveraging that.

# 6.	Creation of ECS Cluster
The creation of ECS Cluster is a key step in scheduling and orchestrating containers in AWS. At this point of time, there are two options for creating ECS Windows Cluster. The first one is to leverage the cloud formation template provided in this link http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html, which provisions ecs cluster and other related resources end-to-end. The second one is to manually create a cluster, set up container instances, ecs agent and other dependencies mentioned in this link http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Windows_getting_started.html. The first option of leveraging Cloud formation template for ecs windows cluster is leveraged in this article.

## **6.1	Custom Cloud Formation template**
The ECS Task definition, ECS Cluster definition and IAM roles are modified in the default cloud formation template mentioned in section 6 is modified to create a custom stack. The customized template is attached below.
![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic15.jpg)
 

## **6.2	ECS Task Definition**
The task definition for running windows container image (ASP.NET MVC) looks like below.

```
"taskdefinition": {
            "Type": "AWS::ECS::TaskDefinition",
            "Properties": {
                "ContainerDefinitions": [{
                    "Name": "aws_ecs_sample",
                    "Cpu": "100",
                    "Essential": "true",
                    "Image": "yourawsaccountnumber.dkr.ecr.ap-southeast-1.amazonaws.com/sundardocker:latest",
                    "Memory": "500",
                    "LogConfiguration": {
                        "LogDriver": "awslogs",
                        "Options": {
                            "awslogs-group": {
                                "Ref": "CloudwatchLogsGroup"
                            },
                            "awslogs-region": {
                                "Ref": "AWS::Region"
                            },
                            "awslogs-stream-prefix": "aws_ecs_sample"
                        }
                    },
                    "PortMappings": [{
                        "ContainerPort": 80
                    }]
                }]
            }
        }


```
## **6.3 ECS Cluster creation and configuration**

The custom cloud formation template created in section 6.1 is validated in the cloud formation designer. The stack creation is initiated by referring to the custom template stored in S3.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic16.jpg)

The desired capacity, instance type, key name, max size, subnetid and vpc id are provided. 

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic17.jpg)

The IAM role for executing cloud formation stack is left with default. Now the stack creation is initiated.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic18.jpg)


The windows container images are generally larger in size. In our case it is around 11 GB. So it takes few minutes to create and configure the entire cluster. The Cloud Formation stack creation is successful. 

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic19.jpg)

It creates all the resources mentioned in the section 6.1.

## **6.4 Verification**
Once the windows container is up and running it can be verified  in the console.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic20.jpg)

Hit the application load balancer url.

![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic21.jpg)

The ASP.NET MVC 4.6 application is rendered in the browser.
 
 ![](https://github.com/sundarnarasiman/Ecswincontainersample/blob/master/screenshots/pic22.jpg)

This shows that the ECS service and ECS task is working fine for windows containers. This completes the migration of windows container to ECS.
