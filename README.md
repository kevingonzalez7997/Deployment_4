# Monitor Resources(4)
October 02. 2023

Kevin Gonzalez

## Purpose:

This project prioritizes real-time monitoring through Amazon CloudWatch for swift issue resolution. We've also fortified security by manually configuring the Virtual Private Cloud (VPC) with private subnets, ensuring overall safety

## Prerequisites:
These will be installed after the EC2 has been configured
- Best practice have the system up to date before installing anything
     - `sudo apt update`
     - `sudo apt updgrade`
- Install "python3.10-venv"
- Install "python-pip"
- Install "python3-pip"
- Install [ngnix](https://www.nginx.com/blog/setting-up-nginx/)
- Install "git-all"



## Steps

### 1. Configure infrastructure
In this deployment, the infrastructure has been created manually. This allows greater control over the environment. Instead of creating subnets, route tables, and availability zones separately, they can all be configured in one step 
- Create “VPC and more”  
- Select 2 AZ
- Select 2 public subnets
- Select 2 private subnets
- For this setup, we are not utilizing NAT gateway or VPC endpoints

### 2. Configure EC2

A few applications will be installed onto the EC2. Since the infrastructure was created manually, the security group will also be configured manually. AWS allows for the configuration of security groups in the creation process of the EC2

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

### 3. Git Clone

Git is a commonly used command-line tool that helps developers track changes in their codebase. Git allows you to create and manage repositories that store the history of your project, including all the changes made to it over time.  Instructions on cloning using git can be found [Here](https://github.com/kevingonzalez7997/Git_Cloning.git)

### 4. Install Jenkins on EC2

- Jenkins is a widespread open-source continuous integration (CI) server. It is used to build, test, and deploy software projects.
- Install Jenkins Server using instructions [Here](https://pkg.jenkins.io/debian/)
- You will need to get a key to set up Jenkins for the first time, run:
     -  `sudo cat  /var/lib/jenkins/secrets/initialAdminPassword`
- After accessing Jenkins through port 8080, install the suggested plug-ins
- Once set up, an additional plugin will be required “Pipeline Keep Running Step”

### 5. Connecting GitHub to Jenkins 

- Most people use GitHub as their repository platform. The code will be pulled from a GitHub repository that we have created as it is a more practical approach.
- create a new item, and select  Multi-branch pipeline
  - Jenkins Credentials Provider:
- Copy and import the Repository URL where the application source code is located
- User Name will be Github user and the password is the generated key in GitHub

### 6. Configure nginx
In the last version, ElasticBean stalk was used to deploy our application. This was replaced by Nginx, a popular open-source web server and reverse proxy server software. It is commonly used to serve web content, handle incoming web requests, and act as a load balancer for distributing traffic across multiple servers or applications. After ngnix has been installed it must also be configured. Since port 80 is already in use for "HTTP", 20 for SSH, and 8080 for Jenkins. 
The application can be accessed through both 5000 and 8000. On port 5000 nginx is running and redirecting to port 8000 where gunicorn is running. This is implemented to separate the web tire and the application tire. That is why in the first edit we swapped out 80 to 5000. For the location, we made it so the link can be reached through port 8000. The data tier is the .json file where the application data is stored.
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

### 7. Installing CloudWatch
Minimizing downtime is one of the main concepts for a successful application. To optimize the application from the previous build a monitor system has been implemented. Since we are using AWS, Cloudwatch was utilized to maintain the native Integration. Another pro is that, it is more cost-efficient as you only pay for the services that you need.

To install run
-`Wget https://amazoncloudwatch-agent.s3.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb` to download the install package file

-`sudo dpkg -i -E ./amazon-cloudwatch-agent.deb` to run the install package

-`cd /opt/aws/amazon-cloudwatch-agent/bin/` cd into app location 

-`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard`
This will launch the setup wizard and allow you to configure in greater detail

-`/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`
Will launch the agent, status can be checked with:
-`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status`
### 8. AWS permissions
- select the current working instance
	- `Actions>Security>Modify IAM role>CloudWatchAgentServer Role`
This is important as this will allow Cloudwatch to have access to the EC2 and access log files

### 9. Setting up Cloudwatch Alerts

While setting up alerts, 3 resources were being selected to monitor. Since the instance we are using has more resources an alarm was set for each one of the CPUs. In addition to further viewing resource usage, an alarm was also set to the RAM.
- Create Alarm
- Search for resources that should be monitored such as "CPU" or "mem"
- Statistic can be set to max to view spiked level more detailed
- Further adjustments can be made such as time intervals and threshold (15% and 5 min were used)

### Observations

CPU.1 Before  
Once the monitor system was attached and deployed a few tests were run to examine resource allocation. First, we need to know what the application's idle CPU usage is. 

After 15 minutes idle did not surpass 5%

CPU.1 After  
Since a multibranch was used, two branches were created with the same code to further test the application resources. The applications were run simultaneously and observed. After a few minutes, the alarms were triggered and emails were delivered.

After two test runs a max CPU usage spike of 52% was recorded.

CPU.2 After  
Since the EC2 t2.Medium has 2 CPUs, both were monitored. After the tests were run we can conclude that the workload was divided equally. Both alerts show highly similar graph patterns. 

RAM Before 
An alarm was also installed to monitor the RAM usage.
Idle ram usage = 25%
With t2.medium resources in RAM have also been increased from 1GB to 4GB

RAM After
numerous test rounds RAM was never affected by more than 5%

Results can be viewed [Here](https://github.com/kevingonzalez7997/Deployment_4/tree/main/results)

### Troubleshooting 
- Some of the issues that are likely to happen are requirements errors
	- Before trying to use a tool, it must be installed correctly beforehand
- Ensure that Jenkins plug-ins are installed for multi-branch ability
- When setting up CloudWatch, if the EC2 created doesn't appear refer to step 8
- when installing Jenkins or any program, the order is important (Especially repo before installing apt)
- When setting up CloudWatch if CPU levels don't show accurately change the metric filter from average to maximum 

### Optimizations
To further optimize the pipeline and minimize downtime, we could consider implementing more alarms. It's important to set alerts for critical CPU levels, such as 85% or higher. Preventing issues before they occur is always preferable, as high CPU levels may lead to the application potentially crashing. Additionally, we can explore horizontal scaling to further distribute traffic. If traffic continues to increase, integrating an instance with higher resources into the application is also an option that can be discussed. 

### Conclusion 
In conclusion, the CPU and RAM monitoring conducted provided valuable insights into the application's performance and resource requirements. Before testing, the application displayed stable and low CPU and RAM usage during idle periods. However, during stress testing with two branches running simultaneously, CPU usage experienced a significant spike, reaching a maximum of 53%. Thanks to the more resourceful EC2 instance with a dual CPU setup, it effectively divided the workload equally, maintaining similar graph patterns for both CPUs. Given that the t2.micro instance has half the resources, it can be concluded that deploying a multibranch configuration would likely overwhelm the EC2 and would not be recommended for this particular project.
