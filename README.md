# 🏗️ AWS VPC Private Subnet Architecture

Implemented a production-grade AWS architecture that deploys applications securely inside a private subnet.

---

## 📐 Architecture Overview

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │ Internet Gateway │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │            VPC               │
              │     (aws-prod-example)        │
              │                              │
              │  ┌─────────────────────────┐ │
              │  │      Public Subnets      │ │
              │  │  ┌──────────┐ ┌───────┐ │ │
              │  │  │   ALB    │ │  NAT  │ │ │
              │  │  │(internet │ │  GW   │ │ │
              │  │  │ facing)  │ │       │ │ │
              │  │  └────┬─────┘ └───────┘ │ │
              │  │       │    us-east-1a/1b │ │
              │  └───────┼─────────────────┘ │
              │          │                    │
              │  ┌───────▼─────────────────┐  │
              │  │     Private Subnets      │  │
              │  │  ┌──────────────────┐   │  │
              │  │  │  Auto Scaling    │   │  │
              │  │  │     Group        │   │  │
              │  │  │ ┌────┐  ┌─────┐  │   │  │
              │  │  │ │EC2 │  │ EC2 │  │   │  │
              │  │  │ │ 1A │  │ 1B  │  │   │  │
              │  │  │ └────┘  └─────┘  │   │  │
              │  │  └──────────────────┘   │  │
              │  │       us-east-1a/1b      │  │
              │  └──────────────────────────┘  │
              └──────────────────────────────────┘
                         ▲
               ┌─────────┴──────────┐
               │   Bastion Host      │
               │  (Public Subnet)    │
               └─────────────────────┘
```

### How traffic flows
- **Inbound:** User → Internet Gateway → ALB → EC2 instances (private subnet)
- **Outbound:** EC2 instances → NAT Gateway → Internet Gateway → Internet
- **Admin SSH:** My machine → Bastion Host → EC2 instances (private subnet)

---

## 🧩 What I Built

| Component | What it does |
|---|---|
| **VPC** | Isolated network for all my resources |
| **Public Subnets (x2)** | Where I placed the ALB and NAT Gateway, one per AZ |
| **Private Subnets (x2)** | Where my application EC2 instances live, one per AZ |
| **Internet Gateway** | Allows public internet traffic into the VPC |
| **NAT Gateway** | Lets private instances reach the internet without exposing their IPs |
| **Application Load Balancer** | Distributes incoming HTTP traffic across my EC2 instances |
| **Auto Scaling Group** | Automatically manages and scales my EC2 instances |
| **Bastion Host** | Jump server I used to SSH into private instances |
| **Security Groups** | Firewall rules I configured per resource |

---

## 📁 Folder Structure

```
aws-vpc-private-subnet-architecture/
│
├── README.md
│
├── architecture/
│   └── architecture-diagram.png
│
├── vpc/
│   └── vpc-setup-notes.md
│
├── compute/
│   ├── launch-template-notes.md
│   └── asg-notes.md
│
├── networking/
│   ├── security-groups.md
│   └── nat-gateway-notes.md
│
├── load-balancer/
│   ├── alb-notes.md
│   └── target-group-notes.md
│
├── bastion/
│   └── bastion-access.md
│
└── app/
    └── index.html
```

---

## 🚀 How I Set This Up

### 1. Created the VPC
I used the **VPC** to create:
- 2 public subnets + 2 private subnets across `us-east-1a` and `us-east-1b`
- 1 NAT Gateway
- Route tables automatically attached to subnets

### 2. Created a Launch Template
- AMI: Ubuntu
- Instance type: `t2.micro`
- Security group with port `22` (SSH) and `8000` (app)

### 3. Created an Auto Scaling Group
- Used the launch template above
- Deployed instances into **private subnets only**
- Desired: 2, Min: 2, Max: 4

### 4. Set up the Bastion Host
I launched a separate EC2 instance in the **public subnet** to use as a jump server to reach my private instances.

### 5. Deployed the App via Bastion

```bash
# Copy my .pem key to the Bastion host
scp -i aws-login.pem aws-login.pem ubuntu@<BASTION_PUBLIC_IP>:~

# SSH into Bastion
ssh -i aws-login.pem ubuntu@<BASTION_PUBLIC_IP>

# From Bastion, SSH into private EC2 (the AMI was different)
ssh -i aws-login.pem ec2-user@<PRIVATE_EC2_IP>

# Deploy a simple Python HTTP server on port 8000
echo "<h1>My First AWS Project</h1>" > index.html
python3 -m http.server 8000
```

### 6. Created the Application Load Balancer
- Scheme: **Internet-facing**, placed in both public subnets
- Security group: port `80` open from `0.0.0.0/0`
- Target group pointing to my EC2 instances on port `8000`

### 7. Tested it
I copied the ALB DNS name and opened it in the browser — the HTML page loaded successfully.

---

## 🔐 Security Decisions I Made

- My EC2 instances have **no public IPs** — completely unreachable directly from the internet
- All user traffic enters only through the **ALB**
- Outbound traffic from private instances goes through **NAT Gateway** so their IPs are never exposed
- SSH access is only possible through the **Bastion Host**, keeping access auditable and controlled
- Security groups follow **least privilege** — only the ports each resource actually needs are open
