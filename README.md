# Hybrid Cloud Network with VPC Peering & Bastion Host

This project demonstrates a secure, multi-network architecture on AWS. It features two custom VPCs connected via VPC Peering, each with public and private subnets. A MySQL server is deployed in a private subnet, and access is managed through a bastion host located in the public subnet of the peered VPC, simulating a hybrid or multi-team infrastructure scenario.

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
This project implements a zero-exposure database architecture following enterprise security best practices. By establishing a private bridge through VPC Peering, the MySQL database is isolated within a restricted production environment, accessible exclusively via a hardened Bastion Host in a separate Management VPC. This design eliminates public internet attack surfaces while preserving secure administrative access—a direct implementation of the AWS Well-Architected Framework's principle of "defense in depth."

## Architecture Diagram
The following screenshots document the step-by-step build, from VPC creation to successful MySQL connection.

## Technical Stack
**Cloud Provider:** AWS (VPC, EC2, IGW, NAT Gateway)

**Networking:** VPC Peering, Custom Route Tables, Security Group Referencing

**Database:** MySQL Community Server (Version 8.0+)

**Security:** SSH Key-Pair Authentication, Least-Privilege IAM/Security Groups

**OS:** Ubuntu 24.04 LTS / Amazon Linux 2023

## Network Infrastructure & VPC Peering
To simulate a professional environment, I utilized non-overlapping CIDR blocks to prevent routing conflicts:

Management VPC (10.0.0.0/16): Houses the public-facing Bastion Host.

Database VPC (192.168.0.0/16): A restricted zone for data assets.

**VPC 1**  
  ![VPC 1](media/image1.png)
- **VPC 2**  
  ![VPC 2](media/image3.png)

The Connectivity Core:

1. VPC Peering: Established a bidirectional peering connection (pcx-xxxx) which allows traffic to route over the AWS private backbone.
  ![Peering Connection](media/image.png)
2. Route Table Logic: Implemented specific routes in both VPCs. The Database VPC’s Private Route Table directs traffic intended for 10.0.0.0/16 through the Peering ID, ensuring the DB can "talk back" to the Bastion.
3. Egress Control: Deployed a NAT Gateway in the public subnet of the Database VPC to allow the MySQL server to pull security patches without exposing port 3306 to the web.

Security Configuration (Defense-in-Depth)
Security is the core pillar of this architecture. I implemented a multi-layered defense strategy that ensures the database remains invisible to the public internet, even if the management layer is compromised.

## Security Configuration
**1. Identity-Based Security Group Referencing**
This is the highlight of the security stack. Rather than using static IP whitelisting (which is brittle and insecure in dynamic cloud environments), I utilized Security Group Nesting.

**Logic**: The MySQL Security Group (Private VPC) is configured to permit inbound traffic on Port 3306 only if the request originates from an instance associated with the Bastion Security Group ID (sg-xxxxxxxx).

Now, This creates a logical "trust boundary." Even if the Bastion Host's private IP changes due to a reboot or scaling event, the database remains accessible to the Bastion—and only the Bastion—without manual intervention.

**2. Zero-Exposure Network Perimeter**
Public Isolation: The MySQL instance has no Public IP address. It exists solely within a private subnet, making it unreachable via the internet gateway.

The Bastion "Jump" Pattern: The Bastion Host acts as the single, hardened entry point. It is the only resource in the entire architecture that "sees" both the public internet (via SSH) and the private database (via the Peering link).

3. Port Hardening & Protocol Scoping
I applied a strict Default-Deny posture for all Network ACLs and Security Groups:
| Resource | Direction | Protocol | Port | Source | Justification |
|----------|-----------|----------|------|--------|---------------|
| Bastion | Inbound | SSH | 22 | My_Admin_IP/32 | Prevents credential brute-forcing from unknown IPs. |
| MySQL | Inbound | TCP | 3306 | sg-Bastion-ID | Restricts DB access to the management tier only. |
| All | Inbound | All | All | 0.0.0.0/0 | **DENIED** - Standard Zero-Trust posture. |

4. Secure Egress via NAT Gateway
The database VPC uses a NAT Gateway for outbound traffic. This allows the MySQL server to reach out for security patches (apt upgrade) without allowing the internet to "reach in." This asymmetric connectivity is critical for maintaining a patched, secure system without increasing the attack surface.

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
Importance:

Network Segmentation: MySQL listens only on the peered VPC's private IP range, not all interfaces

Defense in Depth: Application-layer filtering complements AWS Security Groups

Lateral Movement Prevention: Even if another instance in the same VPC is compromised, it cannot connect to MySQL unless it resides in the 192.168.0.0 range

Zero Trust Implementation: No single point of failure - both network and application layers enforce access control


## Validation & Connectivity Testing
- **Ping Test:** Verified network reachability between instances (if ICMP allowed).
- **SSH Tunneling:** Successfully used the bastion host to SSH into the private instance.
- **MySQL Remote Connection:** From the bastion host, successfully logged into the MySQL server running on the private instance in the other VPC, proving the VPC peering and security group rules work as intended.

## Real-World Application
This architecture is commonly used for:
- **Secure Database Access:** Developers or applications running in a different VPC (or on-prem via VPN) can securely access a database without exposing it to the internet.
- **Multi-Tier Applications:** Separating web servers (in public VPC) from database servers (in private VPC) across different environments.
- **Merged Networks:** Connecting two separate teams or departments (each with their own VPC) to share resources privately.
- **Disaster Recovery:** Establishing private connectivity between a primary and backup infrastructure.

## Future Enhancements
- Automate the entire setup using **Terraform** or **AWS CloudFormation**.
- Implement **AWS Systems Manager Session Manager** to eliminate the need for a public bastion host and manage SSH keys.
- Add an **Application Load Balancer** in the public VPC to distribute traffic.
- Enable **VPC Flow Logs** for network monitoring and troubleshooting.
- Configure **MySQL replication** for high availability across VPCs.

---
**Note:** The screenshots referenced above (e.g., `media/image1.png`) are stored in the `media/` folder of this repository to maintain a clean and organized documentation structure.
