# Deploy Docker container using AWS Fargate
**Created on 13-Jan-2020**

**AWS Fargate** provides a way of running your apps without having to manage servers or scaling clusters. It is part of Amazons Elastic Container Service (ECS). It is the perfect choice for developers that want to run apps with low traffic without spending too much on overpowered EC2 instances. But it will also be able to scale up if there would be higher traffic during some points in the day.

**Amazon Elastic Container Service (ECS)** is a highly scalable, fast, container management service that makes it easy to run, stop, and manage Docker containers on a cluster.

  - Comparable to Kubernetes, Docker Swarm, and Azure Container Service 
  - Runs your containers on a cluster of Amazon EC2 (Elastic Compute Cloud) virtual machine instances pre-installed with Docker
  - Handles installing containers, scaling, monitoring, and managing these instances through both an API and the AWS Management Console.
  - It comes with excellent integration with other AWS services.

# Pre-requisites for running containers on AWS Fargate

  - **ECR Repo** to store our docker images to create the containers 
  - **Cluster** to run services
  - **Task-Definition** which defines what container to run and how to run it in a service. It represents your application
  - **Service** defines the minimum and maximum **tasks** from one **task definition** run at any given time, autoscaling, and load balancing
  - **Load Balancer** to point traffic from the web to our service in the cluster

# Steps to deploy container on ECS:
# 1. Create an ECR Repository
Open the AWS console and go to the service Elastic Container Repository and create a new repo. This is where you will store your docker images that will run in your Fargate cluster.

![Create ECR Repo](https://github.com/nimishajn/dsights-work/blob/master/images/create-ecr-repo.png)

# 2 Steps to push docker image in ECR
- Authentication to AWS:
    Open terminal with administration privileges and enter the following commands:
    ```
    aws configure
    Access key: ****
    Secret key: ****
    ```
    The region name and output format information are not mandatory.
    The data above can be found from the IAM service on AWS console management

- Log in to AWS elastic container registry:
    Use the get-login command to log in to AWS elastic container registry and save it to a text file (see below):

    ```
    aws ecr get-login --region ap-south-2 > text.txt
    ```

- Authenticate Docker to AWS elastic container registry
  Replace the aws account id provided into the text file saved previously and specify the password:
    
    ```
    docker login -u AWS https://<aws_account_id.dkr>.ecr.ap-south-1.amazonaws.com
    Password: *****
    Login_AWS
    ```

- Download the Docker image from Docker hub:
    Use the pull command to download the  image:

    ```
    docker pull <image-name>:6.6
    >> Pull Complete
    ```
- Tag the docker image with you ECR repo URL
    ```
    docker tag <image-name>:<tag> <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com/dsights:<version> 
    ```
    Replace the aws_account_id by your account id

- Push the image into Amazon ECR
    Use the push command to move the centos image into Amazon elastic container registry ECR:

    ```
    docker push <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com/dsights:<version> 
    ```
- Verify your ECR Repo for the image by going to AWS console -> selecting ECR from the menu

# 2. Create Cluster
Create a cluster to run your applications in. 
- Go to **ECS in AWS console** 
- Click **Create Cluster** to get started from scratch. Choose **Networking only** as we will be running our image on Fargate
- Enter the name you want your cluster to have. Check the option of creating a Virtual Private Cloud (VPC) which will give us access to a range of local ip-addresses and subnets.
- **Click “Create” and wait while AWS generates your cluster. When it’s done go back to the ECS-console to view your cluster**

![Create Cluster](https://github.com/nimishajn/dsights-work/blob/master/images/create-ecr-repo.png)

# 3. Create a Task Definition
The task definition defines what docker image to run and how to run it.
- Click on **Create Task Definitions** in ECS
- When creating a new Task Definition you will get the choice of launch compatibility. Choose **Fargate** and click next.
- Name your task and select a task role. I just selected the default IAM role **ecsTaskExecutionRole**
- Scroll down and find the **add container** option. Enter a name for the container and the path of your docker image in ECR repository. **Set the port to the same port that your docker container exposes.**
- Click add container and continue working on the Task Definition. **Set your preferred task memory and task cpu**
- **Press “Create” and you are all set to create a service to run the Task Definition**

![Create Task Definition](https://github.com/nimishajn/dsights-work/blob/master/images/create-task-definition.png)

# 4. Create a Service in Cluster
- **Enter your Cluster, Click Create under the Services tab**
- Once again you will be given the choice of Fargate vs EC2. And once again you will **select Fargate**
- Name your service and select **how many tasks** you want to run. If you enter 2, it will start two instances of your container.
- In the next step we need to **choose a VPC (Virtual Private Cloud) and subnets** which we already have generated in previous step. Select the two subnets available.
- Next up is the **Security group for the cluster**. Click “Edit” and and set a custom TCP-port if your container uses anything other than port 80. For now we will keep it open from anywhere and go back to adjust the source when we have a load balancer set up. Name it so that you can recognise it later.
- Next up we need to **Define a load balancer**. Choose **Application load balancer** from the service configuration and go to the EC2 Console to **Create an http-load balancer**
    - Name the new load balancer and keep most as is. Check the boxes for the availability zones you have available for your VPC. Click next and ignore the HTTPS security warning.
    - In Configure Security Groups we will create another security group. This will decide what port gets to access your load balancer and from where. I choose basic HTTP as it should be accessible from the web.
    - In the Configure routing, we just need to define a Target for the load balancer. The target is where the load balancer sends traffic.
- Now you can add the load balancer to your service and select the container to load balance.
- **Disable service discovery**. We will not be using that now. We will be setting up routing in Route 53 later on.
- Click on next step and do not select any auto scaling features now. **Review and create the service**. If you go back to ECS you will now see that the service is trying to start the amount of tasks you defined in your service.
- Go to **EC2 Console and select Security Groups** from the left menu. **Find the security group we created for the ECS service and edit it. In the  security group inbound rules, add the load balancers Group ID to the source field of port 8050 (port which our container exposes)**. This makes sure that only the load balancer can access the ECS-service on that port.
- The Service is now created and it will now spin up the docker container

![Create Service](https://github.com/nimishajn/dsights-work/blob/master/images/create-service.png)

**Wait for the container to start, It may take few minutes to start and move from PENDING state to RUNNING state**

![Container status](https://github.com/nimishajn/dsights-work/blob/master/images/verify-container-status.png)

# 5. Now to access the application running in the docker container,
- Go to load-balancer in AWS console, copy its DNS name
- Go to web-browser and paste <LB DNS name>:<80>  
- This should open your application running inside the docker container running on the AWS Fargate Cluster.

