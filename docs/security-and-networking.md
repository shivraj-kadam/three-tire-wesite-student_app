# Security and networking

## VPC and subnets

A VPC is a logically isolated network in AWS. A subnet is a range of IP addresses in a VPC and belongs to one Availability Zone.

- **Public subnet:** typically has a route to an Internet Gateway.
- **Private application subnet:** does not have a direct route to an Internet Gateway; backend instances may use controlled outbound access.
- **Private database subnet:** used by the RDS DB subnet group and not intended for direct public access.

The route table—not the subnet's name—determines its internet routing behavior.

## Internet Gateway and NAT Gateway

An Internet Gateway enables internet routing for resources with appropriate public addressing and route-table configuration.

A NAT Gateway allows resources in private subnets to initiate outbound connections, for example to download packages or access public Git repositories, without making those instances directly reachable for new inbound internet connections. NAT Gateways cost money, so review the need and cost for a student project.

## Inbound and outbound rules

- **Inbound:** traffic entering a resource.
- **Outbound:** traffic leaving a resource.

Security groups are stateful: response traffic for an allowed connection is automatically permitted.

Recommended traffic path:

| Resource | Inbound access | Reason |
|---|---|---|
| ALB | TCP 80 for temporary HTTP, or TCP 443 for HTTPS, from intended clients | Public API entry point |
| Backend EC2 | TCP 8080 from ALB security group only | Prevent direct access to application port |
| RDS | Engine-specific DB port from backend security group only | Limit database access to the application |

Restrict SSH to a trusted source IP or use Systems Manager Session Manager when configured.

## IAM roles

An IAM role grants temporary permissions to an AWS workload. An EC2 instance can receive role credentials through an instance profile, avoiding long-lived AWS access keys stored on the instance.

Use least privilege: grant only the AWS actions and resources the application actually needs. An IAM role is not the same as a database username/password and does not automatically enable IAM database authentication.

## Database credentials and secrets

- Never put database credentials, access keys, private keys, or tokens in GitHub.
- Never place database credentials in frontend JavaScript; browser code is visible to users.
- Prefer Secrets Manager or Systems Manager Parameter Store with narrowly scoped IAM permissions.
- Do not share real student data in screenshots or demonstrations.
- If a secret has ever been committed, removing the file alone may not be enough: rotate the secret and follow the repository's secret-removal process.

## RDS security

- Set RDS public access to **No** for this design.
- Place RDS in private database subnets.
- Allow inbound database traffic only from the backend security group.
- Use the correct database port for the selected engine.
- Enable backups and appropriate monitoring for data that matters.
- Keep database credentials out of the repository and logs.

## HTTPS, CORS, and mixed content

HTTPS encrypts traffic between the browser and the endpoint. Configure an issued certificate on the ALB HTTPS listener for the API domain. For an HTTPS frontend, the browser should call an HTTPS API; an HTTPS page calling an HTTP API can be blocked as mixed content.

CORS is a browser policy that controls whether JavaScript from one origin may read responses from another origin. An origin includes the scheme, hostname, and port. Allow the exact deployed frontend origin and required methods/headers rather than allowing every origin permanently.

## S3 frontend and HTTPS

An S3 static website endpoint supports HTTP only. For a public HTTPS site, a common design uses CloudFront with an S3 origin, a suitable certificate, and appropriate origin access configuration.

## Cost and shutdown safety

Potential charges include EC2, EBS, ALB, NAT Gateway, RDS, public IPv4 addresses, data transfer, and snapshots. Stopping EC2 does not stop all related charges.

Before stopping or deleting resources:
- Check whether an ASG will relaunch a stopped instance to maintain desired capacity.
- Confirm whether the ALB, NAT Gateway, and RDS are still needed.
- Back up required data and configuration.
- Delete only resources you understand and no longer need.
- Review AWS Budgets and the Billing console.
