![decagon arch](https://user-images.githubusercontent.com/30019242/185866702-96e254d4-abbf-4813-9c39-ce8ff669047e.png)

# Documentation: #

API Gateway for exposing this service and it help with the security capability
WAF – Web application Firewall to protect against DDOS attack
Amazon Incognito – To authenticate users using token-based authentication
ECS Fargate Cluster – This is used for deploying our microservices using ECS Fargate
RDS – managed relational database (Postgres DB)
AWS ECR Repository
Application load-balancer 
Cloud watch alarms for monitoring
S3 Bucket- /upload API resource is integrated with AWS S3 for any static documents



We will also set up a monitoring system for failure/error or issues in our application

We need to set up a VPC with 2 private and 2 public subnets, and we need to use a bastion host for accessing our Database obviously we need to protect our internal database. This is a security measure.

# Create Posgres RDS instance #
1.	Login to AWS Console 
2.	Choose your preferred regions, like us-east-1
3.	Go to RDS service and create a Postgres Database (non-public with in your VPC with private subnets) from there.
4.	Create the schema inside Database server 
Create ECS Fargate cluster
“Fargate cluster” needs to be created first, under which services can be deployed inside containers.
1.	Login to AWS Console 
2.	Choose your preferred region, like us-east-1
3.	Go to ECS Service Page.
4.	Click on “Create Cluster” button.
5.	Select “Networking only” and click “Next”.
6.	Provide a name like “ECS-fargate-cluster-demo”.
7.	Don’t Select “Create VPC” as we will be using existing VPC.
8.	Select the “CloudWatch Container Insights” check box and click create.
# Create AWS ECR Repository #
ECR repository is a private docker registry. You can upload your docker images inside ECR .  Search ECR  on the page where you have  ECR Click on create repository and provide the name “test1" of the repository here 
1.	Click on “create repository” button.
Upload your docker image to AWS ECR
Time to upload docker image in ECR repository created above.
We have created the ECR repository for the docker image. 
Get the push commands to upload these image to your ECR repo.
a. Go to your ECR repository console.
b. Select the repository and click on “View push commands”.
c. It will open a popup and will show all the commands you need to push the image to ECR.
d. Push test1 image in test1 repository 
Create Application Load Balancer
When we have multiple containers of each service, application load balancer will redirect the traffic to least used node and this way it will make sure load is always balanced between each container holding the same service.
# Let’s create one Application load-balancer now.#
1.	Login to AWS Console 
2.	Go to EC2 Console and select the desired region where you have created your VPC above.
3.	Click on “Load Balancers” from the left menu.
4.	Click on “Create Load Balancer” button.
5.	Click “create” for “Application Load balancer”.
6.	Fill the form with the below information
a. Scheme: Internal (This is because we will be exposing all services using API gateway not directly via load balancer.)
b. IP address type: ipv4
c. Load Balancer Protocol: 80
d. VPC: Select your created VPC.
e. Availability Zones: Select all available zone and select “public subnets” only.
7.	Click on “Next: Configure Security Settings” button.
a. As we are using listener port as 80 and not 443 (with ssl), we will not get the option to setup security configurations. In production you should always use 443 as port with ssl and configure these settings.
8.	Click on “Configure Security Groups” button.
a. Assign a security group: “Create a new security group”
b. Security group name: Give any name. Like: “ecs-fargatel-elb-grp”
9.	Click on the “configure routing” button.
Let’s create “Target Group” 
a. Name: ecs-fargate-test1-tg
b. Target type: IP
c. Keep all setting default except Path for “Health Check”.
Path = /health-check
10.	Click on “Next: Register Targets” button (As we have not deployed any service yet, we don’t have any target. We will setup this later).
11.	Click on “Next: Review” and hit “Create”.
This will create the “Application load balancer” in some time along with one target group “ecs-fargate-test1-tg”
We have added health check path as “/health-check”. “Target group” keeps hitting this path in regular intervals defined in health-checks to get the health of the service. We should create “/health-check” path in each service just for this purpose. Target group will get 200 as response code once it hits this endpoint for any service deployed in containers.
Deploy images in ECS Fargate as containers 
We need to deploy the pushed images to create containers now.
Let’s understand and create the below three IAM roles before we create any container.
1.	Task Role: This role is used to provide access to your services deployed in ECS containers to communicate with other AWS services. “AmazonECS_FullAccess” policy is required for this role to work.
2.	Task execution role”: This role is required by tasks to pull container images and publish container logs to Amazon CloudWatch on your behalf. “AmazonECSTaskExecutionRolePolicy” policy is required for this role to work.
3.	ECS Auto Scale Role: This role will be used to allow ECS to auto-scale ECS tasks based on load. “AmazonEC2ContainerServiceAutoscaleRole” policy is required for this role to work.


# Create Task Definitions #
1.	Login to AWS Console 
2.	Go to ECS Console and select the desired region where you have created your VPC above.
3.	Click on “Task Definitions” from the left menu.
4.	Click on “Create new Task Definition” button.
5.	Select “Fargate” as launch type.
6.	Click on “Next Step” and fill the form with below information.
a. Task Definition Name: ecs-fargate-demo1-td
b. Task Role: This is the IAM role we have created above.
c. Network Mode: awsvpc
d. Task execution role: This is the IAM role we have created above.
e. Task memory (GB): 0.5 GB (This is the maximum memory your task can consume).
f. Task CPU (vCPU): 0.25 vCPU (This is maximum CPU units your task can consume).
7.	Click on “Add Container”.
a. Container name: “ecs-fargate-test1-container”
b. Image: ECR repository URL. You can get the URI from ECR repository you have created above.
c. Memory Limits (MiB): Soft limit: 100 MiB
d. Port mappings: 80
e. Leave all settings as default and hit “Add”.
8.	Click “Create”.
This will create the task definition for the image 
Create Service
1.	Select your created “Task Definition” above.
2.	Select Action “Create Service”.
3.	Fill the form like below:
a. Launch type: Fargate
b. Service name: ecs-fragate-test1-service
c. Number of tasks: 2
d. Minimum healthy percent: 50
e. Maximum percent: 100
4.	Click on “Next step”
5.	Fill the form like below
a. Cluster VPC: Select your VPC.
b. Subnets: Select private subnets (Instance under this subnet can’t be accessed from outside directly).
c. Security groups: Click edit and create a new security group
1. Security group name: “ecs-fargate-containter-sg”
2. Inbound rules for security group: Add the new custom rule for port 80 and add “Application load balancer” security group we have created above as source.
3. Hit save to create security group.
d. Auto-assign public IP: Disabled.
e. Load balancer type: “Application Load Balancer”.
f. Load balancer name: ecs-fatgate-elb.
g. Click on “Add to load balancer”.
1. Production listener port: Select the listener port we have created above.
2. Target group name: ecs-fargate-test1-tg
h. Uncheck “Enable service discovery integration”
6.	Click on “Next Step”.
7.	Service Auto Scaling: Configure Service Auto Scaling to adjust your service’s desired count.
a. Minimum number of tasks: 2
b. Desired number of tasks: 2
c. Maximum number of tasks: 4
d. IAM role for Service Auto Scaling: ECS Auto scale IAM role we have created above
e. Scaling policy type: Step scaling
f. Execute policy when: Create new Alarm
f. Scale out
1. Alarm name: ECS_Fargate_Container_Scaleout_CPU_Utilisation
2. ECS service metric: CPUUtilisztion
3. Alarm threshold: Minimum of CPUUtilisztion > 70 for 1 consecutive periods of 5 mins.
4. Scaling action: Add 1 Tasks when 70 <= CPUUtilisztion
g. Scale in
1. Alarm name: ECS_Fargate_Container_Scalein_CPU_Utilisation
2. ECS service metric: CPUUtilisztion
3. Alarm threshold: Minimum of CPUUtilisztion < 50 for 1 consecutive periods of 5 mins
4. Scaling action: Remove 1 Tasks when 50 > CPUUtilisztion
8.	Click on Next Step and Create the service.
Let’s understand the above configurations.
1.	Minimum and Maximum healthy percent:
a. This will configure the number of tasks should be present at any given time.
b. During deployment, ECS will first bring down the number of tasks defined via “Minimum healthy percent” and start the new tasks with new “task definition version” and once the health check of “Target group” is passed it will bring down the rest of the containers and start new one based on “Maximum healthy percent “. This way your service will always be available at any given time.
2.	Security groups: We have created a security group for the tasks and allowed only “Application load balancer” to communicate with them
3.	Auto-assign public IP: This way we made sure that all our containers will not be accessible outside network.
4.	 Service Auto Scaling: In case of CPU utilisation of the container is more than 70% for continues 5 mins, then it will add one more task. In case CPU utilisation of the container is less than 50% for continues 5 mins, then it will remove one task. But the minimum and maximum number of tasks will always be maintained.
This way we have deployed our services in ECS clusters. 
we have made sure that the task will register itself with ALB while creating. But we have not yet setup the API calls directions to service based on the path in case of scalability if we need to add another service in future.
If one service makes a call like “http://<ALB NAME>/test1” it should reach to test1 service and if call is like “http://<ALB NAME>/test2” then it should reach to test2 service.
1.	Go to EC2 console and click on “Load Balancers”.
2.	Select your created ALB and go to “Listeners” tab.
3.	Click on “View/edit rules”
4.	Click on “+” sign and Insert a rule below.
a. Add Condition: Path
1. is /test1

5.	Click ok
6.	Add Action below
a. Forward to
b. Select Target Group: ecs-fatrgate-test1-tg
7.	Click ok.
 
# Expose API endpoints via API gateway #
AWS API Gateway is managed service for creating and publishing APIs with security and scale. API Gateway is capable of handling hundreds of thousands of concurrent requests.
We will expose our services as Rest API via “API Gateway”. Our services are hosted inside VPC and can’t be accessed via internet currently. We need to use “API Gateway” private integration to expose our private endpoints via “API Gateway”.
Create Network Load Balancer
1.	Login to AWS console of EC2 on the same region where you have created VPC.
2.	Select “Load Balancers” from the left menu.
3.	Click on “Create Load Balancer” button.
4.	Fill the details like below
i. Name: “ecs-fargate-nlb”
ii. Scheme: internal
iii. Load Balancer Protocol: TCP: 80
iv. VPC: Select your VPC.
v. Availability Zones: Select all the available zones
vi. Subnets: Select all the public subnets we have created
5.	Click “Next: Configure Security Settings”
6.	Click “Next: Configure Routing”.
7.	Create the target group like below
i. Name: ecs-fargate-nlb-group
ii. Target type: IP
iii. Protocol: TCP
iv. Port: 80 (On this port ALB is listening).
8.	Click “Next: Register Targets”.
9.	We will add ALB IP addresses later.
10.	Click “Next: Review” and hit create to create the NLB (network load balancer).
Setup the Integration between NLB and ALB
API gateway will send the traffic to NLB and NLB will route the traffic to ALB. We need to setup this communication. To do so, we need to find the IP addresses of both ALB and NLB.  
1.	Add the ALB IP addresses in NLB target group we have created above with name “ecs-fargate-nlb-group”.
2.	Add the IP addresses of NLB in security group of ALB we created above as inbound rule on port 80.
 
# Create VPC Links #
We need to create “VPC Links” in API Gateway to send traffic to NLB, created above.
1.	Login to API Gateway console.
2.	Click on “VPC Links” option in left menu.
3.	Click create and fill the required details like below:
a. Select “VPC link for REST APIs” and click create.
b. Name: ecs-fargate-vpc-link
c. Target NLB: Select the NLB created above.
d. Click create to create the VPC link
Create Rest API
1.	Login to API Gateway console.
2.	Select “APIs” from left menu.
3.	Click on “Build” for “Rest API”.
4.	Select “Rest” and “New API”.
5.	Give a name like “ECS-fargate-api”.
6.	Endpoint Type: Regional
7.	Click create.
8.	Click on “Actions” and select “Create Resource”.
9.	Give the name as “test” and click on “Create Resource”.
10.	Select the resource and Click on Actions.
11.	Click on “Create Method”.
12.	Select “POST” from drop-down and click “ok” next to it
a. Select “Integration Type” as “VPC Link”
b. Select “Use Proxy Integration”
c. Method: POST
d. VPC Link: Select the “VPC Link” we have created above.
e. Endpoint URL: http://<DNS Name of ALB>/test/
f. Hit Save
13.	Select the resource “test” and Click on Actions.
14.	Click on “Create Resource”.
a. Resource Name: {test_id}
b. Resource Path: {test_id}
c. Hit “Create Resource”.
15.	Select the resource “{test_id}” and Click on Actions.
16.	Click on “Create Method”.
17.	Select “GET” from drop down and click “ok” next to it.
a. Select “Integration Type” as “VPC Link”
b. Select “Use Proxy Integration”
c. Method: GET
d. VPC Link: Select the “VPC Link” we have created above.
e. Endpoint URL: http://<DNS Name of ALB>/test1/{test_id}
f. Hit Save
Let’s Deploy the API and test the API
1.	Select Actions and select “Deploy API”
2.	Deployment stage: [New Stage]
3.	Stage name: Test
4.	Hit Deploy to deploy the API.

 
 
