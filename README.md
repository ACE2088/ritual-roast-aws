# Ritual Roast - Highly Available AWS Web Application

## 📌 Project Overview
Ritual Roast is a containerized 3-tier web application deployed on AWS. The primary goal of this Capstone Project was to design, deploy, and troubleshoot a secure, scalable, and highly available cloud infrastructure avoiding hardcoded credentials and public database exposure.

## 📐 High-Level Design (HLD)
![High Level Design Architecture](images/HLD_Architecture.jpg)
*Architecture diagram detailing the flow of traffic from Route 53 through the Application Load Balancer to ECS Fargate containers, with a securely isolated RDS backend.*

![Working Application Proof of Life](images/working-app-ALB.jpg)
*Application successfully handling traffic through the Application Load Balancer.*

---

## 🏗️ Architecture & Networking

The foundation of this project is a robust, custom VPC designed for security and high availability across multiple Availability Zones.

![VPC Resource Map](images/VPC-resourcemap.jpg)
*Custom VPC Architecture showing public subnets for the ALB and private subnets for compute and database resources.*

* **Compute:** Amazon ECS (Fargate) for serverless container orchestration (Docker).
* **Networking:** Amazon VPC, Public/Private Subnets, NAT Gateway, Application Load Balancer (ALB).
* **Database:** Amazon RDS (MySQL) isolated in private subnets.

---

## ⚙️ Orchestration & Load Balancing

The application backend and frontend are orchestrated by **Amazon ECS on AWS Fargate**, eliminating the need to manage underlying EC2 instances.

![ECS Cluster Active](images/web-app-cluster-services-active.jpg)
*ECS Cluster running Fargate tasks seamlessly.*

Traffic is distributed using an Application Load Balancer, which performs continuous health checks and routes traffic to the appropriate Target Groups (Next.js frontend and Flask backend).

![ALB Resource Map](images/ALB-Resourcemap.jpg)
*ALB Resource Map demonstrating listener rules and target group routing.*

---

## 🔒 Security & Best Practices Implemented

To simulate a production-grade environment, strict security boundaries were enforced:

### 1. Database Isolation via Security Groups
The RDS instance is deployed in a Private Subnet with no direct internet access. The Database Security Group is configured to accept inbound traffic on port 3306 **only** from the ECS containers' Security Group.
![RDS Security Group](images/database-security-group-inbound-rules.jpg)

### 2. Secret Management & IAM Roles
Zero credentials are hardcoded in the application source code. The containers retrieve permissions and secrets dynamically using an **IAM Task Execution Role** during the container startup.
![ECS Task Execution Role](images/ecs-task-execution-role.jpg)

---

## 🛠️ Troubleshooting & Debugging (Real-World Scenario)

During the deployment phase, I encountered a critical issue where the backend service became unresponsive. This is a breakdown of my troubleshooting process:

**1. The Symptom:**
Users attempting to access the application were met with a `503 Service Temporarily Unavailable` error thrown by the Application Load Balancer.

![503 Error Symptom](images/working-app-alb-error503.jpg)
*ALB returning a 503 error, indicating it could not route traffic to healthy backend targets.*

**2. The Investigation:**
* **Target Groups:** I checked the ALB Target Groups and saw the ECS tasks were failing the health checks and being constantly drained and restarted by the Auto Scaling group (Task Flapping).
* **ECS Events:** The ECS service events confirmed tasks were stopping with a non-zero exit code.
* **CloudWatch Logs:** I bypassed the load balancer and went directly to the container logs in **Amazon CloudWatch** to find the application-level error.

**3. The Root Cause & Resolution:**
Inside CloudWatch, I found the exact traceback. The Flask application was throwing a `MySQL Error 1045: Access denied for user`. 

![CloudWatch Root Cause](images/flask-container-logs-error-1045.jpg)
*CloudWatch log stream exposing the database authentication failure.*

**Fix:** The backend container was failing to retrieve the correct database credentials from AWS Secrets Manager due to a misconfigured environment variable mapping in the ECS Task Definition. Once the IAM permissions and environment variables were aligned, the container successfully connected to RDS, passed the ALB health checks, and the 503 error was resolved.