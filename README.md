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
The goal was to build a controlled network environment where a private MySQL instance is accessible from a public instance in a **different VPC**, without direct internet exposure. I designed two isolated VPCs, established a VPC peering connection, configured route tables and NAT gateways, and used a bastion host to securely reach the database. This setup mimics real-world requirements for separating environments (e.g., dev/test vs. production) or connecting services across teams while maintaining strict security.

## Architecture Diagram
The following screenshots document the step-by-step build, from VPC creation to successful MySQL connection.

### VPC Creation
Two custom VPCs were created to host the public and private resources separately.
- **VPC 1**  
  ![VPC 1](media/image1.png)
- **VPC 2**  
  ![VPC 2](media/image3.png)

### Subnet Design
Each VPC contains both public and private subnets for layered access.
- **Public Subnet**  
  ![Public Subnet](media/image4.png)
- **Private-Public Subnet (within Private VPC)**  
  ![Private-Public Subnet](media/image5.png)
- **Private Subnet**  
  ![Private Subnet](media/image6.png)

### Internet Gateways & NAT
Internet access is managed per VPC.
- **Internet Gateway (IGW) for Public VPC**  
  ![IGW](media/image7.png)
- **Private VPC IGW (for NAT/outbound)**  
  ![Private IGW](media/image8.png)

### Route Tables
Custom route tables control traffic flow, including peering connections.
- **Private VPC Public RT**  
  ![Private VPC Public RT](media/image9.png)  
  ![Routes](media/image10.png)
- **Private VPC Private RT**  
  ![Private VPC Private RT](media/image12.png)  
  ![Routes](media/image13.png)
- **Public VPC Public RT**  
  ![Public VPC Public RT](media/image14.png)  
  ![Routes](media/image15.png)  
  ![Additional routes](media/image16.png)

### NAT Gateway
Allows private instances to initiate outbound traffic without being publicly accessible.
- **NAT Gateway Setup**  
  ![NAT Gateway](media/image17.png)  
  ![NAT Details](media/image18.png)  
  ![NAT Association](media/image19.png)

### VPC Peering
The core connection linking the two VPCs.
- **Peering Connection Established**  
  ![Peering](media/image20.png)  
  ![Peering Details](media/image21.png)

### Route Edits for Peering
Traffic is directed across the peering link via route table updates.
- **Private VPC Public RT - Peering Route**  
  ![Private VPC Public RT Peering](media/image22.png)
- **Public VPC Public RT - Peering Route**  
  ![Public VPC Public RT Peering](media/image23.png)
- **Private VPC Private RT - Peering Route**  
  ![Private VPC Private RT Peering](media/image24.png)

### Security Groups
Fine-grained access control.
- **Public SG (for Bastion)**  
  ![Public SG](media/image25.png)
- **Private SG (for MySQL)**  
  ![Private SG](media/image26.png)

### EC2 Instances
- **Bastion Host (Public VPC)**  
  ![Bastion Host](media/image27.png)  
  ![Bastion Details](media/image28.png)
- **MySQL Server (Private VPC)**  
  ![MySQL Server](media/image29.png)  
  ![MySQL Details](media/image30.png)

## Technical Stack
- **Cloud Provider:** AWS
- **Core Services:** VPC, EC2, VPC Peering, NAT Gateway, Internet Gateway, Route Tables, Security Groups
- **Database:** MySQL Server (Community Edition)
- **Access Method:** SSH Tunneling / Bastion Host
- **Client Tools:** MySQL Client, OpenSSH

## Network Infrastructure & VPC Peering
- **Two VPCs:** Created with non-overlapping CIDR blocks.
- **Subnets:** Each VPC has a public subnet (for bastion/NAT) and a private subnet (for database).
- **Internet Gateway:** Attached to public subnets for internet access.
- **NAT Gateway:** Deployed in the public subnet of the private VPC to allow the MySQL instance to download updates/patches.
- **VPC Peering:** A peering connection links the two VPCs, enabling private IP communication.
- **Route Tables:** Updated in all relevant subnets to route traffic to the peered VPC’s CIDR block via the peering connection.

## Security Configuration
- **Security Groups:**
  - **Bastion SG:** Allows inbound SSH (port 22) from your trusted IP only.
  - **MySQL SG:** Allows inbound MySQL (port 3306) **only** from the Bastion Host's private IP or security group.
- **Network ACLs:** Left at default (allow all) for this lab, but could be further restricted.
- **Bastion Host:** Acts as a jump box; the MySQL instance has no public IP and is inaccessible directly from the internet.

## Database Setup & Remote Access
1.  **MySQL Installation on Private Server:**
    - Connected to the private instance via the bastion host.
    - Updated the system and installed MySQL server.
    - ![MySQL Install](media/image34.png)  
      ![MySQL Status](media/image35.png)  
      ![MySQL Service](media/image36.png)  
      ![MySQL Secure Installation](media/image37.png)  
      ![MySQL Running](media/image38.png)
2.  **Configuration Change:**
    - Modified the `bind-address` in `/etc/mysql/mysql.conf.d/mysqld.cnf` to `0.0.0.0` to allow remote connections (from the bastion).
    - ![Change Bind Address](media/image40.png)  
      ![Restart MySQL](media/image42.png)
3.  **Access from Bastion Host:**
    - From the bastion, installed the MySQL client.
    - Connected to the private MySQL server using its private IP.
    - ![Bastion MySQL Client Install](media/image43.png)  
      ![MySQL Connection Command](media/image44.png)  
      ![MySQL Login Success](media/image45.png)  
      ![MySQL Show Databases](media/image46.png)

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
