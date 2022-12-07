# Multitier_application_with_loadbalancer_using_docker_swarm
## Description
This is a multi tier application which cache the data collected from ipgeolocation site (https://ipgeolocation.io/) through API request. It utilises docker swarm clustering to obtain geolocation of the user’s IP address. The application can be utilized by testers as well as regular users. This is a three-tier system in which front end marks the first tier where the user can request for the geolocation details of the IP address. The API marks the second tier which connects with the Elasticache (Redis database) to check whether the requested data is cached or not. If the data is not available in Elasticache (Redis database) it connects to the geolocation server to collect the IP details. The Redis database is the third tier where the geolocations collected earlier are cached. The testing team can directly access the 2nd tier with the API URL and this helps them to have a detailed view of the data collected.

![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/final_multi.jpg)

In the architecture the red line marks the three tier application for the users and green line marks the two tier application for the developer or testing team. The main advantage is time efficiency of the project compared to ipgeolocation. The Elasticache (Redis DB) uses key value pair and it stores data in memory whereas in ipgeolocation the MySQL uses tables or HDD to save data, this provides the upper hand for the application in retrieveing data faster.
## Services utilised
Docker Swarm

AWS Services

 a. ALB (Appliation Load Balancer
  
 b. Secret Manager
  
 c. IAM (Policy & Role)
  
 d. Elasticache (Redis DB)
  
 e. EC2
  
  f. VPC


## Stage 1:
    
Create 3 EC2 instances using Amazon Linux  [1 Master, 1 Frontend Server(worker1) and 1 API Server(worker2) ] and the Database server as Amazon ElastiCache[Redis Database]. In this project Master controls the other two worker instances.  
## Stage 2:

Set hostname for the created instances using the following commands:
```
$ sudo hostnamectl set-hostname master
$ sudo hostnamectl set-hostname worker1 # Used as the frontend 
$ sudo hostnamectl set-hostname worker2 # Used as the API server
```
(Logout and relogin to those servers to see the changed hostname)

## Stage 3:
    
Log on to the servers, install docker service and append the login user(ec2-user) under the "docker" group, which is created automatically after the installation of docker service. Check the code below:

```
$ sudo yum install docker -y
$ sudo usermod -a -G docker ec2-user
$ sudo systemctl restart docker.service
$ sudo systemctl enable docker.service
```


## Stage 4:
Create Amazon ElastiCache[Redis] as the Database server, the following are the steps to create ElastiCache:
    
Amazon console--> Select "Amazon ElastiCache"-->ElastiCache --> Redis clusters --> Create --> select "Configure and create a new cluster" --> Cluster mode select disable
name and description given as  "api-redisdb" --> Location "AWS Cloud" --> port:6379 -->Node type: cache.t2.micro --> Subnet groups "choose existing subnet group"--> select default VPC--> Next --> create a security groups or select a security group (if already exists), make sure that the security gruop should be opened the required port like [6379(Redis port),22(SSH Port),8080(App port),80(HTTP), 443(HTTPS), etc] --> Tags [Name:"api-redisdb"] --> Next and Create 

Click on the created Redis cluster(api-redisdb) and under the "Cluster details" and copy the same "Primary endpoint" of "api-redisdb" (eg:  "api-redisdb.v5qvym.ng.0001.aps1.cache.amazonaws.com:6379")

##  Stage 5:  
Create an account in "https://app.ipgeolocation.io" and copy the API Keys to access the ipgeolocation service. In order to secure the API Keys, we can use the Secrets Manager service of AWS. Below are the steps to be followed to store a secret in AWS Secrets Manager. Go to AWS Secrets Manager-->Secrets-->Store a new secret--> Click on "Other type of secret" --> Key/value pairs and click
```
"plain text" -->
{
"ipgeolocation-key":"2cf24ea375f54abe9f0e2d507174811e"
}
```
Note: Replace "2cf24ea375f54abe9f0e2d507174811e" with your secret

next --> secretname and description given as "ipgeolocation-secret"--> Tags[Name:ipgeolocation-secret]-->Next,Next and Store.
    
## Stage 6:
Create a restricted custom policy to access the secret stored in the AWS Secrets Manager. Create a IAM Role using this custom policy and attach the role to the workers ("worker1"  and  "worker2"). Following are the steps to follow the same:
    
Create a policy in IAM named as "Redis-Custom-Policy", with the following permissions:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "secretsmanager:*",
                "cloudformation:CreateChangeSet",
                "cloudformation:DescribeChangeSet",
                "cloudformation:DescribeStackResource",
                "cloudformation:DescribeStacks",
                "cloudformation:ExecuteChangeSet",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcs",
                "kms:DescribeKey",
                "kms:ListAliases",
                "kms:ListKeys",
                "lambda:ListFunctions",
                "rds:DescribeDBClusters",
                "rds:DescribeDBInstances",
                "redshift:DescribeClusters",
                "tag:GetResources"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:secretsmanager:ap-south-1:744353488626:secret:ipgeolocation-secret-YqwbJP"
        },
        {
            "Action": [
                "lambda:AddPermission",
                "lambda:CreateFunction",
                "lambda:GetFunction",
                "lambda:InvokeFunction",
                "lambda:UpdateFunctionConfiguration"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:lambda:*:*:function:SecretsManager*"
        },
        {
            "Action": [
                "serverlessrepo:CreateCloudFormationChangeSet",
                "serverlessrepo:GetApplication"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:serverlessrepo:*:*:applications/SecretsManager*"
        },
        {
            "Action": [
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::awsserverlessrepo-changesets*",
                "arn:aws:s3:::secrets-manager-rotation-apps-*/*"
            ]
        }
    ]
}

```
Note: "arn:aws:secretsmanager:ap-south-1:744353488626:secret:ipgeolocation-secret-YqwbJP" is the 'arn' of secret you have created in stage5

Now go to AWS IAM Service --> Choose Roles session --> Create role and choose Trusted entity type as "AWS service" and Use case as "Ec2" --> and select "Redis-Custom-Policy" policy --> next --> Role name and description as "Redis-Custom-Policy" Tags[Name: "Redis-Custom-Role"] and create role ["Redis-Custom-Role"]


## Stage 7:
Attach the role ["Redis-Custom-Role"] to the workers [worker1 and worker2]
    
Go to AWS instances --> select instance "worker1" and go to "security" session, "modify IAM role" and choose "Redis-Custom-Role" role and update IAM role.

Go to AWS instances --> select instance "worker2" and go to "security" session, "modify IAM role" and choose "Redis-Custom-Role" role and update IAM role


## Stage 8: 
Logon to the master instance and initiate docker swarm mode
```
$ docker swarm init
```
Now the master generate a token. If you want to add workers into this cluster you should run the "join token" to the respective worker instances. A sample "join token" as mentioned below:
```
docker swarm join --token SWMTKN-1-4lmvooss85lrfy9dx40jgd45wuxd190ptbdohtgvt26kb6fl47-6e8k4u2qzq4fbcgnxji8kkyr2 172.31.0.183:2377
```
[If you wish to see the generated toke for references, you may run the command "docker swarm join-token worker" in the master instance]

Now the cluster has been created and you can create containers in the cluster containing a master instance and two worker instances.
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-1.png)
## Stage 9: 
If you wish to restrict the container creation in master instsance, we can "pause" the availability of master. 
    
Steps to set master as "pause" mode to avoid the container as mentioed below:
 
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-2.png)
Now the containers will be created in the worker instances, because they are in active availability 
and the availability of master is set to pause.

## Stage 10: 
Create an oevrlay network for the communication between the containers 

![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-3.png)

## Stage 11: 
Adding labels to the workers from the master instance. Now we create containers based on 
         labelled constraints. 
        
Assign label "frontend" for "worker1" and label "api" for "worker2":
```
[ec2-user@master ~]$ docker node update --label-add app=frontend worker1
worker1
[ec2-user@master ~]$ docker node update --label-add app=api worker2
worker2
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-4.png)
If you wish to see the label of the workers we given in stage 11, you need to inspect the
worker nodes:
    
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-5.png)
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-6.png)

## Stage 12: 
Now we will create containers for Frontend service and API service 

Creating two replicas of frontend container in worker1 having label "frontend"(The applicatiomn port is 8080 and we donot expose the port in this container) 
```
[ec2-user@master ~]$ docker service create \
--constraint node.labels.app==frontend \
--name ipgeolocation-frontend \
--replicas 2 \
--network ipgeolocation-net \
-e API_SERVER="ipgeolocation-api" \
-e API_SERVER_PORT="8080" \
-e APP_PORT="8080" \
fujikomalan/ipgeolocation-frontend:latest
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-7.png)
Creating two replicas of API container in worker2 having label "api"(The applicatiomn port is 8080 and we donot expose the port in this container) 
```
[ec2-user@master ~]$ docker service create \
--constraint node.labels.app==api \
--replicas 2 \
--name ipgeolocation-api \
--network ipgeolocation-net \
-e REDIS_PORT="6379" \
-e REDIS_HOST="api-redisdb.v5qvym.ng.0001.aps1.cache.amazonaws.com" \
-e APP_PORT="8080" \
-e API_KEY_FROM_SECRETSMANAGER="True" \
-e SECRET_NAME="ipgeolocation-secret" \
-e SECRET_KEY="ipgeolocation-key" \
-e REGION_NAME="ap-south-1" \
fujikomalan/ipgeolocation-api-service:latest
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-8.png)
Note: In the above commands we mentioned some environment variables. They are:
   1. REDIS_HOST="api-redisdb.xxxxxx.ng.xxxx.aps1.cache.amazonaws.com" ,It is the endpoint of 
     "Amazon ElasticCache" that we created in stage:4
   2. SECRET_NAME="ipgeolocation-secret"  , it is the name of secret we have created in the stage:5
   3. SECRET_KEY="ipgeolocation-key" ,is the key of secret we have created in the stage:5
   4.  REGION_NAME="ap-south-1" ,is the region where the secrets are placed in the AWS

To listout the services that we created using the command:
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-9.png)
```
[ec2-user@master ~]$ docker service ls

ID             NAME                     MODE         REPLICAS   IMAGE                                          PORTS
lvk6lmuluhnp   ipgeolocation-api        replicated   2/2        fujikomalan/ipgeolocation-api-service:latest   
ty1txjuh6yg9   ipgeolocation-frontend   replicated   2/2        fujikomalan/ipgeolocation-frontend:latest  
```    
    If you want to see the replicas created in the workers, use the following command:
```
[ec2-user@master ~]$ docker service ps  ipgeolocation-frontend

ID             NAME                       IMAGE                                       NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
s60lpq004rcz   ipgeolocation-frontend.1   fujikomalan/ipgeolocation-frontend:latest   worker1   Running         Running 45 seconds ago             
d4uvazo2v550   ipgeolocation-frontend.2   fujikomalan/ipgeolocation-frontend:latest   worker1   Running         Running 45 seconds ago   
    
[ec2-user@master ~]$  docker service ps  ipgeolocation-api

ID             NAME                  IMAGE                                          NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
wwrkdme7l6yq   ipgeolocation-api.1   fujikomalan/ipgeolocation-api-service:latest   worker2   Running         Running 8 minutes ago             
pkgcdznv9w2v   ipgeolocation-api.2   fujikomalan/ipgeolocation-api-service:latest   worker2   Running         Running 8 minutes ago
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-10.png)



## Stage 13: 
In this stage we will create seperate NginX containers for each workers that act as the load balancers for
        workers. 
        
Here the NginX container will be created in master instance as global mode of docker swarm, 
so that the NginX container will be running on the workers that are associated with the cluster. 

subsection1: 
In this work we have created a SelfSigned Certificate and download the key file and certificate into the master machine, and changed its name. URL for getting SelfSigned Certificate: https://getacert.com/ 
```
[ec2-user@master ~]$ wget https://getacert.com/ca/certs/swarmproject-2022-12-07-065011.pkey
[ec2-user@master ~]$ wget https://getacert.com/ca/certs/swarmproject-2022-12-07-065011.cer
[ec2-user@master ~]$ sudo mv swarmproject-2022-12-07-065011.cer trs.cer
[ec2-user@master ~]$ sudo mv swarmproject-2022-12-07-065011.pkey trs.key
[ec2-user@master ~]$ ls -l
total 8
-rw-rw-r-- 1 ec2-user ec2-user 1456 Dec  7 06:50 trs.cer
-rw-rw-r-- 1 ec2-user ec2-user 1679 Dec  7 06:50 trs.key
```
The renamed certificate and key file as shown below:
    

![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-11.png)

subsection2: 
In Docker we can bind-mount directories from local machine to a container. But in docker swarm it is not possible.So that we will be creating secrets and mount these secret files to the NginX container.
    
Now we are creating two secrets for the key and certificate file. These secrets are in encrypted form while stored or in transit. 
        
```
[ec2-user@master ~]$ docker secret create my_key trs.key
ppu00mnbw5zb4mz89acpgv53b
[ec2-user@master ~]$ docker secret create my_cert trs.cer
ez2lgwdk4ppu6ldg57afv6fe0
[ec2-user@master ~]$ docker secret ls
ID                          NAME      DRIVER    CREATED          UPDATED
ez2lgwdk4ppu6ldg57afv6fe0   my_cert             18 seconds ago   18 seconds ago
ppu00mnbw5zb4mz89acpgv53b   my_key              46 seconds ago   46 seconds ago
[ec2-user@master ~]$ 
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-12.png)

subsection3: 
Creating default configuration file for the NginX container:
```
[ec2-user@master ~]$ cat default.conf 
upstream backend {
                server ipgeolocation-api:8080;

        }
upstream backend1 {

server ipgeolocation-frontend:8080;
        }

server {

    
listen 443 ssl;
 ssl_certificate          /run/secrets/my_cert;
 ssl_certificate_key     /run/secrets/my_key;
server_name frontend-geoloc.trsinfotech.link;
   location / {
                proxy_redirect  off;
                proxy_pass http://backend1;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }

   }
server {
    
listen 443 ssl;
 ssl_certificate         /run/secrets/my_cert;

 ssl_certificate_key     /run/secrets/my_key;
server_name api-geoloc.trsinfotech.link;
   location / {
                proxy_redirect  off;
                proxy_pass http://backend;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }

   }
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-13.png)

subsection4:
Save the "default.conf" file in Docker Swarm  service configs. This will help us to avoid the overhead of bind-mounting to the configuration file to the container service.
```
[ec2-user@master ~]$ docker config create my-conf default.conf
qax1tm12eochrvle58hcw5wul
```
In the above command "my-conf" is the name of the config file and "default.conf" is the file to be read it from.

```
[ec2-user@master ~]$ docker config ls
ID                          NAME      CREATED          UPDATED
qax1tm12eochrvle58hcw5wul   my-conf   19 seconds ago   19 seconds ago
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-14.png)
[If we want to remove config use the command: "docker config rm my-conf"]

subsection5:

Creating nginx container in global mode and share the SSL certificate, key file and "default.conf" to the NginX container.
 ```
[ec2-user@master ~]$ docker service create \
      --mode global \
      --network ipgeolocation-net \
      --name nginx \
      --secret my_key \
      --secret my_cert \
      --config source=my-conf,target=/etc/nginx/conf.d/default.conf,mode=0440 \
      -p 443:443 \
      nginx:latest 
 ```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-15.png)
If you wish to see the log details of the container, use the command: "docker service logs nginx" 
 ```
[ec2-user@master ~]$ docker service ps nginx
ID             NAME                              IMAGE          NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
h8cjt5n454ix   nginx.6aye7izex048glarctprnwij6   nginx:latest   worker1   Running         Running 2 minutes ago             
idd6kl2ljdli   nginx.ewj20v164r8z3wxe4n6sc6pez   nginx:latest   worker2   Running         Running 2 minutes ago             
```
![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/swarm-pic-9.png)
    
## Stage 14: 
If you wish to run multiple workers, we should place a load balancer for distributing the traffic to the 
workers. In this work we will create an Application Load Balancer do the same.

########## Creating Target Group ##########

Go to EC2 --> Target groups-->Create target group--> Choose target type: select "Instances" -->Target group name "Docker-swarm-tg" protocol"HTTPS" and Port"443"--> choose default VPC --> Health check protocol "HTTP" --> Health check path"/" --> Advanced health check settings [success code: 200-499] Tags[Name:Docker-swarm-tg]--> Next

Here,
choose first EC2 instance "worker1" and "Ports for the selected instances" 
set as "443" click as "Include as pending below"

Choose first EC2 instance "worker2" and "Ports for the selected instances" 
set as "443" click as "Include as pending below"

Then create target group

########## Create Load Balancer Using This Target Group ##########

Go to AWS, Load balancer--> EC2-->Load balancers --> Select load balancer type -->Application Load Balancer[create]--> loadbalancer name"Docker-Swarm-ALB" -->Scheme: choose "Internet-facing"[Then only everyone can access]--> Choose default VPC Mappings: select "ap-south-1a" and "ap-south-1b"--> Security groups: select "freedom"[by default it is "default" SG]--> Under Listeners and routing: Listener protocol "HTTPS" and port "443" -->Default action:choose the target group "Docker-swarm-tg" --> under Secure listener settings: 
Security policy: ELBSecurity policy-2016-18 & under Default SSL/TLS certificate: "From ACM" and choose "trsinfotech.link" Tags[Name:Docker-Swarm-ALB] and create alb.

Note: Where "trsinfotech.link" is a public domain that was created under route53 and uploaded
      a SelfSigned certificate for the domain.
        
How to import certificate in AWS Certificate Manager ?
Reference URL: https://docs.aws.amazon.com/acm/latest/userguide/import-certificate-prerequisites.html
Then choose the created ALB "Docker-Swarm-ALB" and go to "Listeners" option--> click on "Add listener"--> Under Listener details:
    Protocol:HTTP and Port:80
Default actions and choose "Redirect"
    Protocol:HTTPS and Port:443 then ADD

########## Go to Route53 and create 2 records ##########

AWS-->Route 53-->Hosted zones--> Select the zone "trsinfotech.link" and create record --> Recordname:"frontend-geoloc" --> A record --> Enable "Alias" --> Route traffic to: "Alias to Application and Classic loadbalancer" -->Choose region as "Mumbai[ap-south-1]" --> Choose load balancer we created "dualstack.Docker-Swarm-ALB-1842325209.ap-south-1.elb.amazonaws.com" and create record.


AWS-->Route 53-->Hosted zones--> Select the zone "trsinfotech.link" and create record --> Recordname:"api-geoloc" --> A record --> Enable "Alias" --> Route traffic to: "Alias to Application and Classic loadbalancer"-->Choose region as "Mumbai[ap-south-1]" -->Choose load balancer we created "dualstack.Docker-Swarm-ALB-1842325209.ap-south-1.elb.amazonaws.com" and create record.

########## Choose the ALB "Docker-Swarm-ALB" ##########


Go to Load Balancer --> Choose the ALB "Docker-Swarm-ALB" --> "Listeners" --> Edit "HTTPS:443" listener and Add rule[click on the + symbol] "Insert rule"--> add condition [Host header-->"frontend-geoloc.trsinfotech.link"] --> Add action[Forward to -->Target group"Docker-swarm-tg"] then Save.

Add another rule:
    
Add rule[click on the + symbol]
"Insert rule"--> add condition [Host header-->"api-geoloc.trsinfotech.link"] --> Add action[Forward to -->Target group"Docker-swarm-tg"] then Save.

## Result
Access Sites:
    
  http://api-geoloc.trsinfotech.link/ip/6.6.6.6 -->
  https://api-geoloc.trsinfotech.link/ip/6.6.6.6
    
  http://frontend-geoloc.trsinfotech.link/ip/8.8.8.8 -->
  https://frontend-geoloc.trsinfotech.link/ip/8.8.8.8
  
## FrontEnd Result from ipgeolocation
  
  ![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/frontend-before-cache.png)
  
  ## FrontEnd Result from ipgeolocation after caching
  
  ![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/frontend-after-cache.png)
  
  ## API Result from ipgeolocation
  
  ![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/api-before-cache.png)
  
  ## API Result from ipgeolocation after caching
  
  ![final_multi](https://github.com/ChinduMohan/Multitier_application_with_loadbalancer_using_docker_swarm/blob/main/Docker_swarm/api-after-cache.png)
  
  
