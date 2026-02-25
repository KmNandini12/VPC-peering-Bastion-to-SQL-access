# Hybrid Cloud Network with VPC Peering & Bastion Host

This project demonstrates a **secure, multi-network architecture** on AWS. It features **two custom VPCs** connected via **VPC Peering**, each with public and private subnets. A **MySQL server** is deployed in a private subnet, and access is managed through a **bastion host** located in the public subnet of the peered VPC, simulating a hybrid or multi-team infrastructure scenario.

## 📋 Table of Contents
- [Project Overview](#project-overview)
- [Architecture Diagram](#architecture-diagram)
- [Technical Stack](#technical-stack)
- [Network Infrastructure & VPC Peering](#network-infrastructure--vpc-peering)
- [Security Configuration](#security-configuration)
- [Database Setup & Remote Access](#database-setup--remote-access)
- [Validation & Connectivity Testing](#validation--connectivity-testing)
- [Real-World Application](#real-world-application)
- [Future Enhancements](#future-enhancements)

## Project Overview
This project implements a **zero-exposure database architecture** following **enterprise security best practices**. By establishing a **private bridge through VPC Peering**, the MySQL database is isolated within a **restricted production environment**, accessible exclusively via a **hardened Bastion Host** in a separate **Management VPC**. This design **eliminates public internet attack surfaces** while preserving **secure administrative access**—a direct implementation of the **AWS Well-Architected Framework's principle of "defense in depth."**

## Architecture Diagram
The following screenshots document the step-by-step build, from VPC creation to successful MySQL connection.

## Technical Stack
- **Cloud Provider:** AWS (**VPC, EC2, IGW, NAT Gateway**)
- **Networking:** **VPC Peering**, **Custom Route Tables**, **Security Group Referencing**
- **Database:** **MySQL Community Server** (Version 8.0+)
- **Security:** **SSH Key-Pair Authentication**, **Least-Privilege IAM/Security Groups**
- **OS:** Ubuntu 24.04 LTS / Amazon Linux 2023

## Network Infrastructure & VPC Peering
To simulate a professional environment, I utilized **non-overlapping CIDR blocks** to prevent routing conflicts:

- **Management VPC (10.0.0.0/16):** Houses the **public-facing Bastion Host**.
- **Database VPC (192.168.0.0/16):** A **restricted zone** for data assets.
**VPC 1**  
  ![VPC 1](media/image1.png)
**VPC 2**  
  ![VPC 2](media/image3.png)


**The Connectivity Core:**

- **VPC Peering:** Established a **bidirectional peering connection** (`pcx-xxxx`) which allows traffic to route over the **AWS private backbone**.
![Peering Connection](media/image.png)
- **Route Table Logic:** Implemented **specific routes** in both VPCs. The Database VPC's **Private Route Table** directs traffic intended for `10.0.0.0/16` through the **Peering ID**, ensuring the DB can "talk back" to the Bastion.
- **Egress Control:** Deployed a **NAT Gateway** in the public subnet of the Database VPC to allow the MySQL server to **pull security patches** without exposing port 3306 to the web.

Security is the **core pillar** of this architecture. I implemented a **multi-layered defense strategy** that ensures the database remains **invisible to the public internet**, even if the management layer is compromised.

## Security Configuration
 ### 1. Identity-Based Security Group Referencing
Rather than using **static IP whitelisting** (which is brittle and insecure in dynamic cloud environments), I utilized **Security Group Nesting**.

- **Logic:** The **MySQL Security Group** (Private VPC) is configured to permit inbound traffic on **Port 3306** only if the request originates from an instance associated with the **Bastion Security Group ID** (`sg-xxxxxxxx`).

This creates a logical **"trust boundary."** Even if the Bastion Host's **private IP changes** due to a reboot or scaling event, the database remains accessible to the Bastion—**and only the Bastion**—without manual intervention.

 ### 2. Zero-Exposure Network Perimeter
- **Public Isolation:** The MySQL instance has **no Public IP address**. It exists solely within a **private subnet**, making it **unreachable via the internet gateway**.
- **The Bastion "Jump" Pattern:** The Bastion Host acts as the **single, hardened entry point**. It is the only resource in the entire architecture that "sees" both the **public internet** (via SSH) and the **private database** (via the Peering link).

The Bastion "Jump" Pattern: The Bastion Host acts as the single, hardened entry point. It is the only resource in the entire architecture that "sees" both the public internet (via SSH) and the private database (via the Peering link).

### 3. Port Hardening & Protocol Scoping

I applied a strict Default-Deny posture for all Network ACLs and Security Groups:

| Resource | Direction | Protocol | Port | Source | Justification |
|----------|-----------|----------|------|--------|---------------|
| Bastion | Inbound | SSH | 22 | My_Admin_IP/32 | Prevents credential brute-forcing from unknown IPs. |
| MySQL | Inbound | TCP | 3306 | sg-Bastion-ID | Restricts DB access to the management tier only. |
| All | Inbound | All | All | 0.0.0.0/0 | DENIED - Standard Zero-Trust posture. |

 ### 4. Secure Egress via NAT Gateway
The database VPC uses a **NAT Gateway** for outbound traffic. This allows the MySQL server to reach out for security patches (`apt upgrade`) **without allowing the internet to "reach in."** This **asymmetric connectivity** is critical for maintaining a **patched, secure system** without increasing the attack surface.
## Database Setup & Remote Access

### Configuration Steps

| Step | Action | Details |
|------|--------|---------|
| 1 | Automated Patching | `sudo apt update && sudo apt install mysql-server -y` (via NAT Gateway) |
| 2 | Bind-Address Hardening | Modified `/etc/mysql/mysql.conf.d/mysqld.cnf` to `bind-address = 192.168.0.0` |
| 3 | Least-Privilege User | `CREATE USER 'admin'@'bastion-private-ip' IDENTIFIED BY 'password';` |

### Why Bind to `192.168.0.0`?

```bash
# Default (localhost only - no remote access)
bind-address = 127.0.0.1

# Configured (accepts connections from peered VPC only)
bind-address = 192.168.0.0  # Private IP range of the peered VPC
```
### **Importance:**

- **Network Segmentation:** MySQL listens **only** on the peered VPC's private IP range, **not all interfaces**.
- **Defense in Depth:** **Application-layer filtering** complements AWS Security Groups.
- **Lateral Movement Prevention:** Even if another instance in the same VPC is compromised, it **cannot connect to MySQL** unless it resides in the **192.168.0.0** range.
- **Zero Trust Implementation:** **No single point of failure**—both network and application layers enforce access control.



## Validation & Connectivity Testing

The integrity of the multi-VPC architecture was verified through a **systematic audit** of **network-layer routing**, **transport-layer security**, and **application-layer authentication**.

### Administrative Access and Key Management

- **Secure Hand-off:** Established a secure transfer of the **RSA-4096 private key** from the local workstation to the management tier.
- **Filesystem Hardening:** Applied a strict **`chmod 600`** policy to the key on the Bastion host, satisfying SSH client requirements for **private key security** and preventing **unauthorized read access** at the OS level.

### Inter-VPC Network Verification

- **Layer 3 Peering Audit:** Confirmed **bidirectional reachability** between the Management VPC (`10.0.1.x`) and the Production VPC (`192.168.2.x`) via **ICMP**. This validates the **VPC Peering lifecycle** and the accuracy of the **custom route table entries**.
- **Transit Routing:** Verified that internal traffic successfully traverses the **AWS private backbone** (via `pcx`) rather than exiting through an **internet gateway**.

### Application and Egress Auditing

- **Service Handshake:** Utilized `nc -zv` to verify **Port 3306 availability**. This confirmed that the **Security Group nesting logic** correctly permitted the Bastion's identity while maintaining a **default-deny posture** for all other traffic.
- **Scoped Authentication:** Successfully established a remote MySQL session using a **non-root administrative user**. The connection was restricted to the **Bastion's private IP** to prevent **credential-based lateral movement**.
- **Asymmetric Egress Test:** Validated the **NAT Gateway path** by executing `sudo apt update` from the isolated database instance. This confirmed that the instance can maintain its **security baseline** (patching) **without possessing a public-facing interface**.


### Connectivity Matrix (Verified States)

| Operational Phase | Source | Destination | Protocol | Status |
|-------------------|--------|-------------|----------|--------|
| Management Entry | Local Admin IP | Bastion (Public) | SSH/22 | SUCCESS |
| Cross-VPC Bridge | Management Subnet | Database Subnet | ICMP | SUCCESS |
| Service Access | Bastion SG | MySQL Server | TCP/3306 | SUCCESS |
| Maintenance Egress | MySQL Server | Ubuntu Repositories | HTTPS/443 | SUCCESS |

## Real-World Application

This architecture reflects **enterprise-grade standards** for securing sensitive data while maintaining operational visibility. Its design is applicable to several **high-stakes domains**:

- **Financial Services (FinTech):** Securely isolating **transactional databases** (MySQL) from public-facing web servers or APIs, ensuring that **PII (Personally Identifiable Information)** is **never directly routable** from the internet.

- **Compliance-Driven Infrastructure:** Meeting strict regulatory requirements (e.g., **SOC2, PCI-DSS, or HIPAA**) by implementing **"Defense-in-Depth"** through **VPC Peering**, **Private Subnets**, and **Bastion-based management**.

- **Multi-Team Environment Isolation:** Connecting **disparate VPCs**—such as a Development VPC and a Production VPC—to allow **controlled resource sharing** without merging entire network stacks.

- **Secure Disaster Recovery:** Establishing **private backhaul connectivity** between a primary region and a secondary backup site for **secure data replication** without exposure to public transit.

## Future Enhancements

To evolve this architecture from a manual reference into a **fully automated, production-ready environment**, the following improvements are planned:

- **Infrastructure as Code (IaC) Migration:** Refactoring the entire VPC, Peering, and Security Group stack into **Terraform (HCL)** or **AWS CloudFormation** to ensure **repeatable** and **version-controlled deployments**.

- **AWS Systems Manager (SSM) Integration:** Transitioning to **SSM Session Manager** to **eliminate the need for a public Bastion Host and Port 22 (SSH)** entirely, enabling **browser-based terminal access** governed by **IAM policies**.

- **Managed Database Migration:** Transitioning from an EC2-based MySQL instance to **Amazon RDS** with **Multi-AZ deployment** for **automated backups**, **patching**, and **high availability**.

- **Enhanced Network Monitoring:** Implementing **VPC Flow Logs** and **Amazon CloudWatch alarms** to audit **inter-VPC traffic** and detect **anomalous connection attempts** at the network layer.

- **Application Load Balancing:** Deploying an **ALB** in the public VPC to distribute traffic to **auto-scaling web tiers**, further decoupling the application layer from the database tier.
---
**Note:** The screenshots referenced above (e.g., `media/image1.png`) are stored in the `media/` folder of this repository to maintain a clean and organized documentation structure.
