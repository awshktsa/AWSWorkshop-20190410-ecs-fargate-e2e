# Startup Workshop Series (2019-04-10) Build a fargate ecs wordpress(or any services) with aurora serverless
## In some real cases, we want to move our application from one single EC2/VM into decoupled pieces. Today we take a common case as example, to move your all-in-one wordpress into a scalable architecture with Application Load Balancer + ECS Fargate + Aurora Serverless. 

For this workshop, we use cloud9 as our working environment:

N. Viginia(us-east-1)
Ohio (us-east-2)
Oregon(us-west-2)
Tokyo(ap-northeast-1)
Singapore (ap-southeast-1)
Ireland (eu-west-1)

* Please be noted, you can skip the cloud9 and paking the docker from your own environment, please be careful to manage your AWS credential for programmetic usage.

### 1. Setup a new IAM key with proper privileges
 - **AWS Console > IAM > user***
 - You can create a temp user for this workshop, or setup for long term dev purpose.
 - Enable the privileges to have *AmazonEC2ContainerRegistryFullAccess* privilege 

### 2. Setup your cloud9 working environment
 - **AWS Console > Cloud9**
 - Create a new cloud9 environment if you never have one
 - Find the AWS setting in preference in the IDE view, and turn off the credential setting 
   “AWS managed temporary credential = off”
 - open a shell tab, and configure your AWS-CLI
 ```> aws configure```
 - and input the AKSK from Step.1 for your workshop usage

### 3.	Build your wordpress docker-image and push to ECR
 - Setup a new directory to build a fargate-wp-docker
 ```> git clone  https://github.com/WordPress/WordPress.git```

### 4.	Change the wp-config.php to setup your wordpress
In general case, sometimes we will configure all the application data within our source code, but this time we modify the code to accept environment variable, which means you can change the application running status with parameter. In this way you can also run your application in different stages/environments.
```
> cd Wordpress
> cp wp-config-sample.php wp-config.php
```

edit wp-config.php within your editor, and replace DB_NAME,DB_USER,DB_PASSWORD,DB_HOST setting with getenv() like following:

```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', getenv('DB_NAME') );

/** MySQL database username */
define( 'DB_USER', getenv('DB_USER') );

/** MySQL database password */
define( 'DB_PASSWORD', getenv('DB_PASSWORD') );

/** MySQL hostname */
define( 'DB_HOST',  getenv('DB_HOST') );
```
For more detail about get env, please check ref: (https://www.php.net/manual/en/function.getenv.php)


### 5.	Create docker file to build up your wordpress
- create a new file *Dockerfile* under Wordpress, same level with wp-config.php

```
FROM richarvey/nginx-php-fpm
COPY . /var/www/html/

WORKDIR /var/www/html
```
Here you can see, we only build up a super simple docker image. We based on a nginx-php-fpm image, and packed the modified php code into target director /var/www/html, which is the standard setting if we want to run our php application with nginx and php-fpm. 

### 6. Create a new ECR repository, and push docker image onto your repository
- **AWS Console > ECR**
- Give a repository name as you like, ex:ecr-wp-docker
- Clike the "View push commands", and copy the commands
- 4 steps: retrive the login, build the docker image, tag your image, and then push to ECR
- Finally you will find an image URI after you push the docker image

### 7. Start to build the sourrounding components for our wordpress in ECS-Fargate
- **AWS Console > VPC > Security Groups**
- Now we need to create 3 Security Groups 
- One for ALB (ex: sg-fargate-alb), ingress port 80 from 0.0.0.0/0
- One for ECS, ingress * from sg-fargate-alb
- One for Aurora (ex:sg-fargate-aurora ), ingress 3306 from sg-fargate-ecs, and also your cloud9. (If you use local desktop, you need to have a jumpbox/bastion to handle this part)

### 8.	Create an Aurora Serverless
- **AWS Console > RDS > Create Database > Aurora > Serverless**
- Setup the security group in sg-fargate-aurora
- Get the DB_HOST value from your aurora console, and remember your $DB_ROOT and password
- Connect into your Aurora from cloud9 shell tab 

```> mysql -h $DB_HOST -u $DB_ROOT -p ```

- create a database user and database for your wordpress
```mysql> create database wordpress default charset=utf8;
mysql> create user ‘wordpress’@’%’ identified by ‘wordpress’;
mysql> grant all privileges on wordpress.* to ‘wordpress’@’%’;
mysql> flush privileges;
```
### 9.	Create Target Group 
- **AWS Console > EC2 > Target Group > Create Target Group*
- Indentify the group name (ex:tg-fargate-ecs)
- Target type=ip
- Now just set with empty host

### 10.	Create ALB alb-fargatre
- **AWS Console > EC2 > Loadbalancer > Create LoadBalancer**
- Pick security group with sg-fargate-alb we created in step.7
- Set traffic to target group tg-fargate-ecs we created in step.9


### 11.	Create ECS Task-definition
After we setup the Fargate preparation, now we start to move our previous work to ECS-Fargate. In this step we define how we run with this ECS task inside.
- **AWS Console > ECS > Task Definition > Create New Task Definition**, 
- Give a name (ex:td-ecs-fargate-wp)
- Use default setting, and create Task execution IAM role by default
- **Add Container and define which docker image you want to run**
- In the container detail, you have to fill the image URI value you get in step.6
- And also you will need to input the port mapping, port=80
- In the container detail, the advanced section you can see environment variable, setup the KEY=Value from step 8, which will looks like following:
```
DB_NAME value wordpress
DB_USER value whateverusernameyousetup-in-step-8
DB_PASSWORD value IhaveAsuperDifficultPassword
DB_HOST value myworkshopaurora.cluster-abcdefghijk.ap-southeast-1.rds.amazonaws.com
```

### 12.	Finally, create ECS Farget
If you heard about ECS, the Elastic Container Service, then you might know that was container service provided by AWS. Before we jumping into service itself, you have to allocate a fargate cluster for your "Serverless" container. Which means you don't need to manage the EC2 clusters for your containers.
- **AWS Console > ECS > Cluster > Create Cluster** give a name (ex:cls-fargate) and type=fargate
- Click into the Farget cluster you created
- See the service tab, and click **create**
- Launch type=Farget
- Task Definition is what we created in Step.11
- Select correct fargate cluster, and pick correct load balancing. 
- Give proper number setting and click **Next**.
- In the Network configuration part, please select the Application Load Balancer we created in Step.10
- In the "Container to load balance", select correct port setting and target group we create in Step.9.
- After all set, create your ECS services, and verify through ALB public address.
