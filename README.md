# AWS VPC High Availability Lab

**Author:** Mohammed Abdul Basit  
**Lab:** Securely Deploying Resources in a VPC using CloudFormation  
**Course:** BEM11 — Deep Dive into Networking on AWS

---

## Overview

A production-grade, highly available, multi-AZ Virtual Private Cloud deployed entirely via AWS CloudFormation. The architecture demonstrates fault-tolerant network design, controlled internet access, and secure EC2 instance management — with zero SSH access. All instance access is performed exclusively through AWS Systems Manager Session Manager.

---

## Architecture

```
                        ┌─────────────────────────────────────────────────────┐
                        │                    VPC  10.0.0.0/16                 │
                        │                                                      │
                        │   ┌──────── AZ-1 ────────┐  ┌──────── AZ-2 ───────┐│
                        │   │                      │  │                     ││
  Internet ──── IGW ────┼───┤  Public 10.0.1.0/24  │  │ Public 10.0.2.0/24  ││
                        │   │  ┌────────────────┐  │  │ ┌────────────────┐  ││
                        │   │  │ Web Server AZ-1│  │  │ │Web Server AZ-2 │  ││
                        │   │  │   (Apache)     │  │  │ │  (Apache)      │  ││
                        │   │  └────────────────┘  │  │ └────────────────┘  ││
                        │   │  NAT Gateway 1        │  │ NAT Gateway 2       ││
                        │   │        │             │  │       │             ││
                        │   ├────────┼─────────────┤  ├───────┼─────────────┤│
                        │   │        ▼             │  │       ▼             ││
                        │   │ Private 10.0.11.0/24 │  │Private 10.0.12.0/24 ││
                        │   │  ┌────────────────┐  │  │ ┌────────────────┐  ││
                        │   │  │ App Server AZ-1│  │  │ │App Server AZ-2 │  ││
                        │   │  │  (SSM only)    │  │  │ │  (SSM only)    │  ││
                        │   │  └────────────────┘  │  │ └────────────────┘  ││
                        │   └──────────────────────┘  └─────────────────────┘│
                        └─────────────────────────────────────────────────────┘
```

---

## Infrastructure Components

| Resource | Count | Details |
|---|---|---|
| VPC | 1 | `10.0.0.0/16`, DNS support + hostnames enabled |
| Public Subnets | 2 | `10.0.1.0/24` (AZ-1), `10.0.2.0/24` (AZ-2) |
| Private Subnets | 2 | `10.0.11.0/24` (AZ-1), `10.0.12.0/24` (AZ-2) |
| Internet Gateway | 1 | Attached to VPC |
| NAT Gateways | 2 | One per AZ — AZ-scoped pattern |
| Elastic IPs | 2 | One per NAT Gateway |
| Route Tables | 3 | 1 public (shared), 2 private (AZ-local) |
| Security Groups | 2 | Web tier, App tier |
| IAM Role | 1 | `AmazonSSMManagedInstanceCore` |
| EC2 Instances | 4 | 2 web (public), 2 app (private) |

---

## Network Design

### CIDR Layout

```
VPC:              10.0.0.0/16    (65,536 IPs)
Public AZ-1:      10.0.1.0/24   (256 IPs)
Public AZ-2:      10.0.2.0/24   (256 IPs)
Private AZ-1:     10.0.11.0/24  (256 IPs)
Private AZ-2:     10.0.12.0/24  (256 IPs)
```

The gap between public (`/1x`) and private (`/11x`) ranges is intentional — it leaves room for future tiers (e.g. a database tier at `10.0.21.0/24`) without renumbering.

### Routing

| Subnet | Route Table | Default Route |
|---|---|---|
| Public AZ-1 & AZ-2 | `PublicRouteTable` | `0.0.0.0/0` → Internet Gateway |
| Private AZ-1 | `PrivateRouteTable1` | `0.0.0.0/0` → NAT Gateway 1 (AZ-1) |
| Private AZ-2 | `PrivateRouteTable2` | `0.0.0.0/0` → NAT Gateway 2 (AZ-2) |

Private subnets have **egress-only** internet access. There is **no cross-AZ NAT dependency** — if AZ-1 fails, AZ-2 private instances still reach the internet through their own local NAT Gateway.

---

## Security Groups

### Web Tier (`vpc-ha-lab-web-sg`)

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 80 | `0.0.0.0/0` | HTTP from internet |
| Inbound | ICMP | All | VPC CIDR | Ping for validation |
| Outbound | All | All | `0.0.0.0/0` | Package updates, SSM |

### App Tier (`vpc-ha-lab-app-sg`)

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | ICMP | All | VPC CIDR | Ping for validation |
| Outbound | All | All | `0.0.0.0/0` | Outbound via NAT GW |

**No inbound SSH (port 22) on any instance.**

---

## EC2 Instances

### Web Tier (Public)

Both instances run Apache HTTP Server installed via CloudFormation UserData. The web page dynamically displays:

- Student name and lab name (injected via CloudFormation parameters)
- Instance ID, Availability Zone, Private IP, and Public IP (fetched from IMDSv2 at boot)

### App Tier (Private)

Minimal instances with no web server. Accessible only via SSM Session Manager. Outbound internet access is available through their AZ-local NAT Gateway for package management and validation.

---

## Access — No SSH

All four instances are managed exclusively via **AWS Systems Manager Session Manager**.

**Why no SSH?**
- No open inbound ports required
- IAM-based authentication (no SSH keys to lose or rotate)
- Full session audit trail
- Works on private instances with no public IP

### Connecting via Session Manager

**Console:**
1. Go to **Systems Manager → Session Manager → Start session**
2. Select an instance and click **Start session**

**CLI:**
```bash
aws ssm start-session --target <instance-id> --region us-east-2
```

Instance IDs are available in the CloudFormation **Outputs** tab after deployment.

---

## Deployment

This stack is deployed via **CloudFormation GitSync**. Pushing changes to this repository automatically triggers a stack update.

### Deployment Configuration (`deployment.yaml`)

```yaml
template-file-path: template.yaml
stack-name: vpc-ha-lab-stack
Parameters:
  VpcCidr: "10.0.0.0/16"
  InstanceType: "t3.micro"
  StudentName: "Mohammed Abdul Basit"
  LabName: "AWS VPC High Availability Lab"
  ProjectTag: "vpc-ha-lab"
```

### Manual Deployment (Alternative)

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name vpc-ha-lab-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-2 \
  --parameter-overrides \
    StudentName="Mohammed Abdul Basit" \
    LabName="AWS VPC High Availability Lab" \
    ProjectTag="vpc-ha-lab"
```

### Stack Outputs

After deployment the following values are available in the CloudFormation Outputs tab:

| Output | Description |
|---|---|
| `WebServer1URL` | HTTP URL for Web Server AZ-1 |
| `WebServer2URL` | HTTP URL for Web Server AZ-2 |
| `WebInstance1Id` | Instance ID for SSM access (Web AZ-1) |
| `WebInstance2Id` | Instance ID for SSM access (Web AZ-2) |
| `AppInstance1Id` | Instance ID for SSM access (App AZ-1) |
| `AppInstance2Id` | Instance ID for SSM access (App AZ-2) |
| `NatGateway1EIP` | Public IP of NAT Gateway AZ-1 |
| `NatGateway2EIP` | Public IP of NAT Gateway AZ-2 |

---

## Validation

### 1. Web Tier — Browser Access

Open both URLs from CloudFormation Outputs. Each page shows the instance ID, AZ, and IP addresses confirming both servers are live and independent.

### 2. Public ↔ Private Connectivity (Session Manager)

Connect to a web instance and ping an app instance:

```bash
ping 10.0.11.x   # App Server AZ-1
ping 10.0.12.x   # App Server AZ-2
```

### 3. Private Instance Outbound Internet (via NAT Gateway)

Connect to an app instance via Session Manager and run:

```bash
# Confirm outbound internet
ping 8.8.8.8

# Trace the path — first hop will be the NAT Gateway
traceroute 8.8.8.8

# Install a package to prove full outbound connectivity
sudo dnf install -y curl
```

---

## Design Decisions

See [design-decisons.md](design-decisons.md) for detailed reasoning on:

- AZ-scoped NAT Gateways vs a single shared NAT Gateway
- AWS Regional NAT Gateway and how it could replace this architecture
- Why SSH is disabled in favour of SSM Session Manager
- Separate route tables per private subnet
- IMDSv2 usage in UserData scripts
- CloudFormation best practices applied

---

## CloudFormation Best Practices Applied

- **No hardcoded AMI IDs** — `LatestAmiId` resolves via SSM Parameter Store to always use the latest Amazon Linux 2023 AMI
- **DependsOn for EIPs** — prevents race condition between EIP allocation and IGW attachment
- **`!Sub` for all names** — `ProjectTag` parameter drives consistent naming across all resources
- **VPC ID exported** — available for cross-stack references via `!ImportValue`
- **Console UI grouping** — parameters grouped by Network, EC2, and Identification in the CloudFormation console
- **Tags on everything** — `Name`, `Project`, `Tier`, and `AZ` on every resource for filtering and cost allocation
