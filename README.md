# Monitor Resources(4)
October 02. 2023

Kevin Gonzalez

## Purpose:

To monitor web server and its resource usage

In the previous build, automation was incorporated into the pipeline with the use of webhook and Elastic Beanstalk CLI. Any update done to Git Hub would get automatically redeployed. This was an improvement but didn't account for monitoring. Through the use of CloudWatch, abnormal levels of CPU usage are reported instantly. In addition, the VCP was configured manually instead of the default. This helps increase security by implementing private subnets.

## Prerequisites:
These will be installed after the EC2 has been configured
- Best practice have the system up to date before installing anything
     - `sudo apt update`
     - `sudo apt updgrade`
- Install "python3.10-venv"
- Install "python-pip"
- Install "python3-pip"
- Install [ngnix] (https://www.nginx.com/blog/setting-up-nginx/)
- Install "git-all"



## Steps

### 1. Configure infrastructure
In this deployment, the infrastructure has been created manually. This allows greater control over the environment. 
- Create “VPC and more”  
- Create 2 AZ
- Create 2 public subnets
- For this setup, we are not utilizing NAT gateway or VPC endpoints

### Configure EC2

For this setup, a few applications will be installed onto the EC2. Since the infrastructure was created manually, the security group will also be configured manually. AWS allows for the configuration of security groups in the creation process of the EC2

-Launch instance
-Instance type: t2.Medium 
	- This is an upgrade in both RAM and CPU from t2.micro
- The keys have been generated before
- In the network setting select the VPC that was created earlier
- Select `public1` subnet that was created earlier
- Select “Auto-assign public IP” This is important as the public IP is what makes the EC2 visible 
This is where AWS allows you to create and configure the instance's security group. There are four different inbound rules that will be configured. By default ssh, port 22 is already added. Continue with port 80,8000,8080
- For port 80 in the type section select HTTP, it should by default select the port
- for port 8000 select custom TCP
- for port 8080 select custom TCP as well
- Launch and connect to instance

### Git Clone

Git is a commonly used command-line tool that helps developers track changes in their codebase. Git allows you to create and manage repositories that store the history of your project, including all the changes made to it over time.  Instructions on cloning using git can be found [Here] (https://github.com/kevingonzalez7997/Git_Cloning.git)

### Install Jenkins on EC2

- Jenkins is a popular open-source continuous integration (CI) server. It is used to build, test, and deploy software projects.
- Install Jenkins Server using instructions [Here](https://pkg.jenkins.io/debian/)
- You will need to get a key to set up Jenkins for the first time, run:
     -  `sudo cat  /var/lib/jenkins/secrets/initialAdminPassword`
- After accessing Jenkins through port 8080, install the suggested plug-ins
- Once set up, an additional plugin will be required “Pipeline Keep Running Step”


###  Connecting GitHub to Jenkins 

- Most people use GitHub as their repository platform. The code will be pulled from a GitHub repository that we have created as it is a more practical approach.
- create a new item, and select  Multi-branch pipeline
  - Jenkins Credentials Provider:
- Copy and import the Repository URL where the application source code is located
- User Name will be Github user and the password is the generated key in GitHub

### configure nginx
After ngnix has been installed it must also be configured.
Run the following to edit 
`sudo nano /etc/nginx/sites-enabled/default`

<pre>
<code>
server {
    listen 5000 default_server;
    listen [::]:5000 default_server;
    # We changed the port from 80 to 5000

    # Also scroll down to where you see “location” and replace it with the text below:
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
</code>
</pre>

### Installing CloudWatch
Minimizing downtime is one of the main concepts for a successful application. In order to optimize the application from the previous build a monitor system has been implemented. Since we are using AWS, Cloudwatch was utilized to maintain the native Integration. Another pro is that, it is more cost-efficient as you only pay for the services that you need.

To install run
-`Wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb` to download install package file

-`sudo dpkg -i -E ./amazon-cloudwatch-agent.deb` to run install package

-`cd /opt/aws/amazon-cloudwatch-agent/bin/`

-`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard`
This will launch the setup wizard 

### AWS permissions

Once the EC2 is running 

- select the current working instance
	- `Actions>Security>Modify IAM role>CloudWatchAgentServer Role` 
### 
