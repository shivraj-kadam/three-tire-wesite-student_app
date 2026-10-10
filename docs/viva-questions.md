# Teacher viva questions and sample answers

Keep answers short and explain them in your own words. Only claim that an optional feature is implemented if you have verified it in the AWS Console.

## Project basics

### 1. What is your project?
It is a Student Registration Web Application deployed on AWS. The frontend is hosted separately, a Java backend runs on EC2 behind an Application Load Balancer, and application data is stored in RDS.

### 2. What is a three-tier architecture?
It separates the presentation tier, application/business-logic tier, and data tier. This helps separate responsibilities and apply different security and scaling controls to each layer.

### 3. What happens when a user submits the registration form?
The browser sends an API request to the ALB or API subdomain. The ALB forwards it to a healthy backend target. The Java backend processes the request, accesses RDS when required, and returns a response to the browser.

## AWS services

### 4. Why did you use Amazon S3?
S3 stores static frontend files without requiring me to manage a web server for those files. For HTTPS hosting, a common production design uses CloudFront with an S3 origin.

### 5. What is Amazon EC2?
EC2 provides virtual servers in AWS. In this project, Ubuntu EC2 runs the Java backend application.

### 6. What is an Application Load Balancer?
An ALB is a Layer 7 load balancer that accepts HTTP/HTTPS requests and routes them to configured targets using listeners and rules. It can route traffic to healthy backend targets.

### 7. What is a target group?
A target group contains the backend targets that receive forwarded traffic. It also defines the target protocol, port, and health-check configuration.

### 8. What does Healthy mean in a target group?
It means the target passed the configured health check. It does not prove every application feature or database query works.

### 9. What is Amazon RDS?
RDS is a managed relational database service. AWS manages much of the underlying database administration, while the application still needs correct schema, credentials, connectivity, and permissions.

## Networking and security

### 10. What is a VPC?
A VPC is a logically isolated network in AWS where I configure IP ranges, subnets, route tables, and network security.

### 11. What is the difference between public and private subnets?
A public subnet generally has a route to an Internet Gateway. A private subnet does not have a direct route to an Internet Gateway. The route table determines the routing behavior.

### 12. Why keep the backend in a private subnet?
It reduces direct exposure to the internet. Users reach the application through the ALB, and the backend security group permits application traffic from the ALB security group.

### 13. Why keep RDS private?
The database should not accept direct public internet connections. The backend connects through the VPC, and the RDS security group restricts which resources can connect.

### 14. What is a security group?
A security group is a stateful virtual firewall attached to a resource or network interface. It controls allowed inbound and outbound traffic.

### 15. What is the difference between inbound and outbound rules?
Inbound rules control traffic entering a resource. Outbound rules control traffic leaving it. Security groups are stateful, so response traffic for an allowed connection is automatically permitted.

### 16. What is a NAT Gateway?
A NAT Gateway lets resources in private subnets initiate outbound internet connections, such as downloading updates, without making those resources directly reachable for new inbound internet connections.

### 17. What is an IAM role?
An IAM role provides temporary permissions to a workload. An EC2 instance can use an instance profile to obtain role credentials instead of storing long-lived AWS access keys.

### 18. Does an IAM role automatically log the application into RDS?
No. IAM database authentication must be supported and enabled for the selected engine, and the database user, IAM policy, token generation, and application configuration must be correct. Otherwise, the application uses securely stored database credentials.

## Compute and operations

### 19. What is an Auto Scaling Group?
An ASG maintains a desired number of EC2 instances and can replace unhealthy instances or scale capacity according to configured policies. I would claim this is enabled only after verifying the ASG and target-group attachment.

### 20. What is a Launch Template?
It is a reusable configuration for launching EC2 instances, including settings such as the AMI, instance type, security groups, storage, IAM instance profile, and user data.

### 21. Why use systemd for the Java backend?
systemd manages the service, supports start/stop/status operations, can start it on boot, and can restart it after failures.

### 22. How do you check whether the backend is running?
I use `sudo systemctl status studentapp`, inspect logs with `sudo journalctl -u studentapp`, and confirm the listening port with `sudo ss -lntp | grep 8080`.

## Web application concepts

### 23. What is CORS?
CORS is a browser security mechanism that controls whether JavaScript from one origin can read responses from another origin. The backend should allow the deployed frontend origin.

### 24. Why can the website load while registration fails?
The frontend may use an old API IP, the ALB may have no healthy target, the listener or port may be wrong, CORS may block the request, or the backend may fail to connect to RDS.

### 25. Why should the frontend call the ALB rather than an EC2 public IP?
The ALB is the stable entry point and can route requests to healthy targets. Using a specific EC2 IP couples the frontend to an individual instance and can break when the instance changes.

### 26. Why use HTTPS?
HTTPS encrypts traffic in transit and helps protect data from interception or modification. The frontend and API should both use HTTPS in the final public deployment.

### 27. What is EBS?
Amazon Elastic Block Store provides block storage volumes for EC2. The root volume commonly stores the operating system and files. EBS storage can continue to incur charges even when an EC2 instance is stopped.

### 28. What is EFS?
Amazon Elastic File System is a managed shared file system that multiple supported compute instances can mount. It differs from EBS, which is typically used as block storage attached to an individual instance.

### 29. What are the main cost considerations?
EC2, EBS, ALB, NAT Gateway, RDS, public IPv4 addresses, data transfer, and snapshots can incur charges. Stopping EC2 does not stop every related charge, so I review Billing and remove unused resources carefully.

## 2–3 minute presentation script

Good morning. My project is a Student Registration Web Application deployed on Amazon Web Services. The objective was to host the frontend, run the backend, and store application data using managed AWS services.

The design has three main tiers. The presentation tier is the static frontend hosted on Amazon S3. The application tier is a Java backend running on Ubuntu EC2. The data tier is Amazon RDS. The Application Load Balancer is the entry point for API traffic and forwards requests to backend targets through a target group.

For networking, the VPC separates public subnets for the load balancer from private subnets for the backend and database. The backend security group allows application traffic from the ALB security group, and the RDS security group allows database traffic only from the backend security group. This reduces unnecessary exposure.

I manage the Java application as a systemd service and use service status, logs, and port checks to troubleshoot it. I also verify the target-group health and ensure the frontend API URL points to the ALB or API domain instead of an individual EC2 IP address.

Finally, I test the registration flow, inspect browser network errors, check backend logs, and verify database persistence where authorized. Improvements for a production deployment include HTTPS for the frontend and API, secure secret storage, monitoring, backups, and automated deployment. Thank you.

## Final advice

Before the viva, know your actual AWS Region, subnet layout, ALB DNS name, listener ports, target-group health-check path, RDS engine, frontend URL, and API URL. Never show passwords, tokens, private keys, or real student records.
