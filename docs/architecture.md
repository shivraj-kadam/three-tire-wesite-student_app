# Architecture and request flow

## 1. Three-tier design

### Presentation tier — Amazon S3

S3 stores static frontend assets such as HTML, CSS, JavaScript, images, and fonts. The browser downloads these files and runs the frontend code.

**HTTPS note:** S3 static website endpoints are HTTP-only. For a public HTTPS website, a typical production pattern is CloudFront in front of an S3 origin, configured with an appropriate certificate and origin access controls.

### Application tier — ALB and Ubuntu EC2

The Application Load Balancer provides a stable entry point for API requests. Its listener accepts traffic and forwards matching requests to a target group. The backend runs on Ubuntu EC2 and, in the discussed setup, listens on TCP port 8080.

The Java service should be managed using systemd so that it can be started at boot and inspected with service and log commands.

### Data tier — Amazon RDS

RDS provides a managed relational database. The backend connects to the RDS endpoint over the VPC network. The database should use private subnets and a security group that allows the database port only from the backend security group.

## 2. Request flow

1. The user opens the frontend URL.
2. The browser downloads static files from S3 (or from CloudFront if configured).
3. The user submits the registration form.
4. Frontend JavaScript sends an API request to the ALB DNS name or an API subdomain that resolves to the ALB.
5. The ALB listener forwards the request to a healthy target in the target group.
6. The backend validates and processes the registration.
7. The backend connects to RDS if database access is required.
8. The backend returns a response, which passes through the ALB to the browser.

The frontend should not call an individual EC2 public IP. Use the ALB DNS name or a DNS name mapped to the ALB.

## 3. Network layout

A common layout is:

- **Public subnets in at least two Availability Zones:** ALB and routes to an Internet Gateway.
- **Private application subnets:** backend EC2 instances; outbound internet access may require NAT or another controlled egress design.
- **Private database subnets:** RDS DB subnet group across Availability Zones; no direct public access.

Subnet names alone do not make a subnet public or private. Routing configuration determines whether a subnet has a direct route to an Internet Gateway.

## 4. High availability and scaling

An ALB can distribute requests across healthy targets. If an Auto Scaling Group is configured, it can launch instances from a Launch Template, maintain desired capacity, and replace unhealthy instances. Confirm that the ASG is actually attached to the target group before claiming that automatic scaling or replacement is implemented.

An RDS deployment can use backups and, if configured, Multi-AZ options. Do not claim Multi-AZ database failover unless it is enabled in the live database configuration.

## 5. Diagram (logical)

```text
                   User / Browser
                         |
                  Frontend website
                    Amazon S3
                         |
                  API request (HTTP/S)
                         |
              Application Load Balancer
                   Public subnets
                         |
                    Target group
                   TCP/HTTP : 8080
                         |
                Ubuntu EC2 / Java API
                 Private app subnets
                         |
                 Database connection
                         |
                    Amazon RDS
                 Private DB subnets
```

This is a logical diagram. Add the exact subnet names, ports, domain, and services from the AWS Console to match the actual deployment.
