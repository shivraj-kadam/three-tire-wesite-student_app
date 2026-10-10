# End-to-end deployment steps

This guide follows the project architecture discussed: S3 frontend, ALB, Ubuntu EC2 Java backend on port 8080, and RDS. AWS Console labels can change. Confirm actual values before running commands.

## Step 1 — Plan the VPC

1. Select the AWS Region.
2. Create or select a VPC with an appropriate IPv4 CIDR range.
3. Create public subnets in at least two Availability Zones for the internet-facing ALB.
4. Create private application subnets for backend instances.
5. Create private database subnets in at least two Availability Zones for the RDS DB subnet group.
6. Record the VPC ID, subnet IDs, route tables, and Availability Zones.

## Step 2 — Configure routing

- Attach an Internet Gateway to the VPC.
- Associate public subnet route tables with the public subnets and add `0.0.0.0/0 → Internet Gateway`.
- Private application subnets should not route directly to an Internet Gateway. If the instance needs to download packages or clone a public repository, provide a NAT Gateway or a controlled alternative.
- Keep database subnets private; do not add a public route to make RDS accessible from the internet.

A NAT Gateway can incur hourly and data-processing charges. Verify dependencies before deleting one.

## Step 3 — Create security groups

Recommended baseline:

- **ALB security group:** inbound TCP 80 for temporary HTTP testing or TCP 443 for HTTPS, from intended clients. For a public website this is commonly `0.0.0.0/0`; restrict it if the site is meant to be private.
- **Backend EC2 security group:** inbound TCP 8080 from the ALB security group only. SSH TCP 22 should be restricted to a trusted admin IP, or use Systems Manager Session Manager where configured.
- **RDS security group:** inbound database port from the backend security group only. The port depends on the database engine (for example, MySQL commonly uses 3306 and PostgreSQL commonly uses 5432).

Avoid opening port 8080 or the database port to all internet addresses.

## Step 4 — Create RDS

1. Select the database engine required by the application.
2. Create a DB subnet group using private database subnets in multiple Availability Zones.
3. Set public access to **No**.
4. Attach the RDS security group with inbound access only from the backend security group.
5. Create the database and user credentials.
6. Record the endpoint hostname, database name, engine, and port. Do not put passwords in this repository.

Store secrets in a secure service such as AWS Secrets Manager or Systems Manager Parameter Store with appropriate IAM permissions, or use the approved secure configuration method for the project.

**IAM database authentication:** An EC2 IAM role does not automatically make every database login work. The selected engine must support IAM database authentication, it must be enabled, and the database user, policy, token generation, and application must be configured accordingly.

## Step 5 — Launch or configure Ubuntu EC2

Use Ubuntu for the backend. Select the backend security group and place the instance in a private application subnet when that matches the architecture. Attach an EC2 IAM role only when the workload needs AWS API permissions.

If using an Auto Scaling Group, configure the Launch Template and ASG to use the intended private subnets and attach the target group to the ASG.

## Step 6 — Install Java, Maven, and Git

Connect to the Ubuntu instance through your approved administration method and run:

```bash
sudo apt update
sudo apt install -y git maven openjdk-17-jdk
java -version
mvn -version
```

Use the Java version required by the repository's build configuration.

## Step 7 — Clone and build the backend

The repository used in the project discussion was:

`https://github.com/mukundDeo9325/CDEC-studentapp.git`

Confirm that this is still the correct repository and that the instance has access. If cloning returns “repository not found,” verify the URL, owner/repository spelling, repository visibility, and permissions.

```bash
git clone https://github.com/mukundDeo9325/CDEC-studentapp.git
cd CDEC-studentapp
ls
find . -maxdepth 3 -name pom.xml
```

Move into the directory containing the backend's `pom.xml` before building:

```bash
mvn clean package
find target -maxdepth 1 -name '*.jar'
```

Use the actual JAR filename and path; do not assume every repository has the same folder layout.

## Step 8 — Run the backend as a systemd service

For a quick local test, the JAR can be run with `java -jar`, but a managed service is more appropriate for a persistent deployment.

Example service file (replace paths and JAR name with the actual ones):

```ini
[Unit]
Description=Student Registration Backend
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/opt/CDEC-studentapp/backend
ExecStart=/usr/bin/java -jar /opt/CDEC-studentapp/backend/target/APP.jar
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Ensure the real files are in the specified path and that the service user can read required configuration. Keep secrets out of the source code and service file where possible.

```bash
sudo systemctl daemon-reload
sudo systemctl enable studentapp
sudo systemctl start studentapp
sudo systemctl status studentapp --no-pager
sudo journalctl -u studentapp -n 100 --no-pager
sudo ss -lntp | grep 8080
```

## Step 9 — Create the target group

1. Create an instance target group using the correct VPC.
2. Use HTTP and port 8080 if the backend serves HTTP on that port.
3. Configure a health-check path that returns a successful response without requiring login. Use the application's actual health endpoint; do not assume a path exists.
4. Register the backend instance, or attach the target group to the ASG so instances register automatically.
5. Wait for the target status to become **Healthy**.

A healthy target proves that the configured health check succeeds; it does not prove every registration/database operation works.

## Step 10 — Create the Application Load Balancer

1. Choose an internet-facing ALB if public users need to access the API.
2. Select at least two public subnets in different Availability Zones.
3. Attach the ALB security group.
4. Create an HTTP listener for a temporary test or an HTTPS listener with an issued ACM certificate for the final setup.
5. Forward the listener or matching path rule to the backend target group.
6. Copy the ALB DNS name for testing.

## Step 11 — Configure Auto Scaling (if used)

- Create a Launch Template with the AMI, instance type, security group, storage, IAM instance profile if required, and repeatable setup.
- Configure the ASG with private application subnets across Availability Zones.
- Attach the target group.
- Set minimum, desired, and maximum capacity according to the project's needs and budget.
- Enable load-balancer health checks if appropriate and configure a sensible health-check grace period.

Do not claim ASG-based replacement or scaling unless these settings are present.

## Step 12 — Configure the frontend API URL

Search the frontend source for any old hard-coded EC2 IP address or port. Set the API base URL to the ALB DNS name or the API subdomain that points to the ALB.

Example only:

```javascript
// Temporary HTTP test only
const API_BASE_URL = "http://YOUR-ALB-DNS-NAME";

// Preferred after a domain and HTTPS listener are configured
const API_BASE_URL = "https://api.your-domain.example";
```

Replace these example values with the actual endpoint. If the project uses environment variables, update the correct build-time variable and rebuild the frontend. Upload the built static files to the correct S3 bucket.

## Step 13 — Configure a domain and HTTPS (if required)

- Create a DNS record for the API subdomain pointing to the ALB using the supported Route 53 alias or DNS provider configuration.
- Request and validate an ACM certificate for the exact hostname.
- The ALB's ACM certificate must be in the ALB's Region. CloudFront certificates must be requested in `us-east-1`.
- Attach the issued certificate to the HTTPS listener and configure an HTTP-to-HTTPS redirect if appropriate.
- For HTTPS frontend hosting, use CloudFront with an S3 origin rather than relying on the S3 static website endpoint.

If certificate validation fails, verify the DNS validation record exactly matches ACM's instructions, check domain ownership and CAA records, and confirm the Region.

## Step 14 — End-to-end test

1. Open the frontend URL.
2. Submit a test registration using non-sensitive data.
3. Open browser Developer Tools → Network and inspect the request URL, status, and response.
4. Confirm the request goes to the ALB/API domain rather than a stale EC2 IP.
5. Check target health and backend logs.
6. Verify database persistence where authorized and available.
7. Check for CORS or mixed-content errors.

See [Testing and troubleshooting](testing-and-troubleshooting.md) for more checks.
