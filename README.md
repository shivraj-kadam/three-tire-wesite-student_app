# AWS Three-Tier Student Registration Web Application

A project documentation repository for presenting the Student Registration Web Application deployed on Amazon Web Services (AWS).

> **Important:** This repository documents the architecture and deployment approach discussed for the project. Confirm every account-specific setting in the AWS Console before presenting. A feature described as a recommendation is not necessarily implemented in the live environment.

## Project overview

The application separates the user interface, application logic, and data storage:

- **Frontend:** static website files hosted on Amazon S3.
- **Traffic routing:** Application Load Balancer (ALB) and a target group.
- **Backend:** Java/Spring Boot application on Ubuntu Amazon EC2, listening on port 8080 as configured in the project.
- **Database:** Amazon RDS, accessed by the backend over private networking.
- **Network:** Amazon VPC with public and private subnets, route tables, and security groups.
- **Optional scaling:** EC2 Launch Template and Auto Scaling Group (ASG), if enabled in the deployed setup.
- **Optional custom domain and TLS:** DNS record to the ALB and an ACM certificate for HTTPS.

## Request flow

1. A user opens the frontend website.
2. S3 serves the static HTML, CSS, JavaScript, and other frontend assets.
3. The user submits the registration form.
4. The browser sends an API request to the configured ALB DNS name or API subdomain.
5. The ALB listener forwards the request to a healthy target in the target group.
6. The Java backend validates and processes the request.
7. The backend reads or writes application data in RDS.
8. The response returns to the browser.

## Documentation

| File | Contents |
|---|---|
| [Architecture](docs/architecture.md) | Components, request flow, and network layout |
| [Deployment steps](docs/deployment-steps.md) | End-to-end setup and verification |
| [Security and networking](docs/security-and-networking.md) | VPC, subnet, routing, security groups, IAM, and secrets |
| [Testing and troubleshooting](docs/testing-and-troubleshooting.md) | Validation checklist and common failures |
| [Teacher viva questions](docs/viva-questions.md) | Simple interview answers and presentation script |

## Project details to confirm before a demo

- AWS Region and VPC CIDR
- Public, application-private, and database-private subnet IDs and Availability Zones
- ALB DNS name and listener ports
- Target group name, backend port, and health-check path
- RDS engine and port (do not publish credentials)
- Frontend URL and API base URL
- Whether ASG, custom domain, ACM certificate, HTTPS, and CloudFront are actually configured

## Security reminders

- Never commit database passwords, AWS access keys, private keys, tokens, or real student data.
- Do not expose RDS directly to the public internet.
- Permit backend port 8080 from the ALB security group rather than from every IP address.
- The frontend must not contain database credentials or AWS secrets.
- An S3 static website endpoint supports HTTP only. For HTTPS, a common approach is CloudFront in front of an S3 origin.

## Scope

This repository is documentation, not the application source code. See the application repository separately for the actual frontend and backend implementation.

## Presentation tip

Demonstrate the working registration flow, then show the ALB target health, EC2 service status, and database configuration with secrets hidden. Explain honestly which optional features are live and which are future improvements.
