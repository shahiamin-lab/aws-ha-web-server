# AWS Cloud Architecture Capstone Report

**Project:** Highly Available Web Server  
**Author:** Amin Shahi  
**Region:** US West (Oregon), us-west-2

## 1. Executive Summary

This project deploys a highly available web server using Amazon EC2, an Application Load Balancer (ALB), an EC2 Auto Scaling group, Amazon CloudWatch, and AWS CloudFormation. The design distributes HTTP requests across EC2 instances in two Availability Zones. Auto Scaling maintains the desired capacity and can replace unhealthy instances. CloudFormation defines the infrastructure so it can be deployed consistently. The application was tested through the public load balancer endpoint.

## 2. Architecture and Service Choices

Internet users send HTTP requests to the Application Load Balancer on port 80. The ALB forwards requests to a target group and performs health checks. The target group routes traffic to EC2 instances in two Availability Zones.

The EC2 Auto Scaling group maintains a desired capacity of two instances, with a maximum capacity of four. A target-tracking scaling policy uses a 50% average CPU utilization target. A CloudWatch alarm monitors whether the number of in-service instances falls below two. CloudFormation provisions the infrastructure from the template in `infrastructure/ha-web-server.yaml`.

EC2 was chosen because it provides control over the operating system and Apache web server. An ALB provides Layer 7 HTTP routing and health checks. Auto Scaling supports instance replacement and capacity adjustments. CloudFormation provides repeatable infrastructure deployment. A container platform such as ECS Fargate could reduce server-management work, while a serverless design could remove instance management, but both would require a different application and deployment design.

## 3. Implementation and Testing

The CloudFormation template creates the load balancer, target group, security groups, launch template, Auto Scaling group, scaling policy, and monitoring alarm. The EC2 startup script installs Apache, obtains instance metadata using IMDSv2, and generates a web page showing the project name, author, instance ID, and Availability Zone.

Testing confirmed that the CloudFormation stack reached `UPDATE_COMPLETE`, two EC2 instances were `InService` and healthy, both target-group targets were healthy, and the ALB returned the expected HTML. The EC2 instance refresh completed successfully. Screenshots and JSON validation outputs are saved in `evidence/`.

One implementation issue encountered was a package conflict involving `curl-minimal`. The startup script was adjusted to install Apache with `dnf install -y httpd` and avoid the unnecessary curl package. The resulting instances served the page successfully.

## 4. Security, Reliability, Performance, and Cost

### Security

The EC2 security group allows web traffic only from the ALB security group, rather than directly from the internet. The launch template requires IMDSv2 for instance metadata access. The ALB currently serves HTTP, so traffic is not encrypted in transit. A production version should use HTTPS with an AWS Certificate Manager certificate, restrict outbound access where practical, and review IAM permissions and logging.

### Reliability and Disaster Recovery

The ALB and EC2 instances span two Availability Zones, reducing dependence on a single zone. Health checks help prevent traffic from being routed to unhealthy instances, and Auto Scaling can replace failed instances. The design does not provide a tested regional disaster-recovery solution and does not claim a specific recovery time objective (RTO) or recovery point objective (RPO). A regional recovery plan would require replicated infrastructure, appropriate data replication if data were added, DNS or traffic failover, and documented recovery tests.

### Performance

The ALB distributes requests across healthy targets, while Auto Scaling can add capacity in response to sustained CPU load. The CPU target of 50% is a scaling threshold, not a guarantee of response time. Further work should include load testing, latency monitoring, and testing scaling behavior under representative traffic.

### Cost Estimate and Optimization

The following is an illustrative planning estimate, not a verified AWS Pricing Calculator result or actual Learner Lab charge. It assumes two small EC2 instances run continuously for approximately 730 hours per month, one ALB runs continuously, traffic is low, and no paid domain or TLS certificate is required. Actual cost depends on the selected instance type, ALB capacity units, public IPv4 charges, data transfer, CloudWatch usage, and regional pricing.

Before production use, enter these assumptions in the AWS Pricing Calculator for US West (Oregon) and record the resulting monthly estimate: two EC2 instances, one Application Load Balancer, expected ALB capacity usage, CloudWatch alarms and metrics, public IPv4 addresses, and estimated outbound data transfer.

Two cost optimizations are to stop or delete non-production resources when they are not needed and to right-size EC2 capacity based on measured CPU and traffic. For production workloads, Savings Plans or Reserved Instances may reduce eligible compute costs when usage is predictable. The two-instance minimum should not be reduced without accepting the corresponding availability trade-off.

## 5. AWS Well-Architected Self-Review

### Operational Excellence
**Strength:** CloudFormation provides repeatable infrastructure deployment, and the project includes saved verification evidence.  
**Improvement:** Add deployment automation, centralized logs, alert notifications, and a documented incident-response procedure.

### Security
**Strength:** The EC2 security group accepts web traffic only from the ALB security group, and IMDSv2 is required.  
**Improvement:** Enable HTTPS, strengthen least-privilege permissions, review outbound rules, and implement security monitoring.

### Reliability
**Strength:** The load balancer and instances span two Availability Zones, with health checks and Auto Scaling.  
**Improvement:** Test Availability Zone failure scenarios and create a documented, tested regional recovery plan with defined RTO and RPO targets.

## 6. What I Would Do Differently

I would add HTTPS and automated deployment earlier, test instance replacement and scaling under load, and establish CloudWatch notifications before final validation. I would also use the AWS Pricing Calculator before deployment and document a regional recovery procedure. These improvements would strengthen security, operational visibility, and cost planning.

## 7. Cleanup

After saving the required screenshots and validating the repository, the CloudFormation stack should be deleted to avoid ongoing charges. The cleanup command and completion check are documented in `README.md`. The project report and saved evidence are intended to remain in the repository after the deployed resources are removed.
