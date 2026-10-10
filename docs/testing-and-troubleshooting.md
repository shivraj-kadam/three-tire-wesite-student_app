# Testing and troubleshooting

## End-to-end checklist

- [ ] Frontend URL loads and assets appear correctly.
- [ ] Frontend API base URL points to the ALB or API subdomain, not an old EC2 IP.
- [ ] ALB listener forwards to the intended target group.
- [ ] Target group reports the backend as **Healthy**.
- [ ] Backend systemd service is active.
- [ ] Java service is listening on the expected port, such as 8080.
- [ ] Backend security group allows that port from the ALB security group.
- [ ] RDS is not publicly accessible.
- [ ] RDS security group permits the engine port only from the backend security group.
- [ ] Registration returns the expected response.
- [ ] Database persistence is checked where authorized.
- [ ] Browser Console and Network panels show no CORS or mixed-content errors.
- [ ] No credentials or personal data appear in code, Git history, or screenshots.
- [ ] Optional features such as ASG, HTTPS, and CloudFront are verified in the console before being described as implemented.

## Useful Ubuntu commands

```bash
# Check service status
sudo systemctl status studentapp --no-pager

# Inspect recent service logs
sudo journalctl -u studentapp -n 100 --no-pager

# Confirm the backend is listening on port 8080
sudo ss -lntp | grep 8080

# Test the local backend (replace / with a known API/health path if needed)
curl -i http://127.0.0.1:8080/

# Check DNS resolution for RDS
getent hosts YOUR-RDS-ENDPOINT

# Test TCP connectivity; replace 3306 with the actual database port
nc -vz YOUR-RDS-ENDPOINT 3306
```

A running systemd service does not guarantee the API or database connection is working. Use logs and actual requests to confirm application behavior.

## Common problems

### Target group is Unhealthy

Check:
1. The backend service is running.
2. The service is listening on the target group's configured port.
3. The health-check path returns a successful status without authentication.
4. The backend security group allows the port from the ALB security group.
5. The target group protocol matches the service protocol.
6. The instance is in the expected VPC and target configuration.

### ALB returns 502 or 503

- **502** can occur if the target closes a connection or returns an invalid response.
- **503** can occur if there are no healthy or usable targets.

Check target health, service logs, target port/protocol, listener rules, and whether the application is actually listening.

### Frontend opens but registration fails

Inspect browser Developer Tools → Network:
- Is the request URL the expected ALB/API domain?
- Is the HTTP status 4xx or 5xx?
- Does the browser report CORS or mixed content?
- Do backend logs show the request?
- Can the backend connect to RDS?

### Frontend still calls an old IP

Search the frontend source and built output for the previous IP/port. Update the correct API base URL or build-time environment variable, rebuild, upload the output to S3, and invalidate CloudFront cache if CloudFront is used.

### CORS error

Configure the backend to allow the exact frontend origin, required HTTP methods, and headers. Do not use an unrestricted wildcard as a permanent fix when credentials or sensitive data are involved.

### Mixed-content error

If the frontend is HTTPS, use HTTPS for the API as well. Configure an issued certificate on the ALB HTTPS listener and point the API subdomain to the ALB.

### EC2 cannot clone the Git repository

Check:
- Repository URL spelling and owner/name.
- Repository visibility and authorization.
- Outbound routing, DNS, and network access.
- Whether a private repository requires authenticated access.

Never paste a personal access token into shared notes, Git history, or a public shell transcript.

### Backend cannot connect to RDS

Check the RDS endpoint hostname, database engine port, database status, credentials or secret configuration, VPC routing, subnet groups, and RDS security-group inbound source. A successful DNS lookup alone does not prove that the database port is reachable.

### ACM certificate stays Pending or fails validation

Check that:
- The domain is under your control.
- The DNS validation record exactly matches the record ACM provides.
- There are no relevant CAA restrictions.
- The certificate covers the exact hostname.
- The certificate is in the correct Region (ALB Region for ALB; us-east-1 for CloudFront).

### Maven build fails

Check the Java and Maven versions, current directory, presence of `pom.xml`, dependency download access, and the first meaningful error in the output.

## Evidence for the teacher

Capture the following screens, hiding secrets and personal data:
1. VPC and subnet layout.
2. Route tables.
3. Security group rules.
4. ALB listeners.
5. Target group health.
6. EC2 service status and logs.
7. RDS configuration showing private access (never credentials).
8. Browser Network tab showing the successful registration request.
9. Successful frontend registration result.

Do not claim a database row was saved unless you actually verified persistence.
