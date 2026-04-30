# Design Decisions & Architecture Notes

## Why Two NAT Gateways?

The single most important HA decision in this architecture is deploying **one NAT Gateway per AZ** rather than one shared NAT Gateway.

### The Risk of a Single NAT Gateway

If you place one NAT Gateway in AZ-1 and both private subnets route through it:
- If AZ-1 suffers a partial failure (NAT Gateway becomes unavailable), all private instances in AZ-2 **lose outbound internet access** even though AZ-2 itself is healthy.
- This defeats the purpose of multi-AZ design.
- You also pay cross-AZ data transfer fees for all AZ-2 → NAT-GW-1 → internet traffic.

### This Architecture's Solution

Each private subnet has its own local route table pointing to its AZ-local NAT Gateway:
- `PrivateRouteTable1` → `NatGateway1` (both in AZ-1)
- `PrivateRouteTable2` → `NatGateway2` (both in AZ-2)

This is called the **AZ-scoped NAT pattern**.

---

## Why No SSH?

Security groups intentionally omit port 22. All access is via **AWS Systems Manager Session Manager**, which:
- Requires no open inbound ports
- Authenticates using IAM roles (not SSH keys that can be lost/stolen)
- Logs all session activity to CloudWatch/S3 (audit trail)
- Works for instances in private subnets with no public IP

The SSM agent on Amazon Linux 2023 communicates **outbound** to SSM service endpoints over HTTPS (port 443). Because private instances have outbound internet access via NAT Gateways, SSM sessions work without VPC Interface Endpoints.

---

## Why Separate Route Tables for Each Private Subnet?

A single shared route table for both private subnets would require routing to a single NAT Gateway, reintroducing the cross-AZ dependency. Separate route tables allow each AZ's private subnet to independently route to its local NAT Gateway.

---

## CIDR Design

```
VPC:              10.0.0.0/16   (65,536 IPs)
Public AZ-1:      10.0.1.0/24  (256 IPs)
Public AZ-2:      10.0.2.0/24  (256 IPs)
Private AZ-1:     10.0.11.0/24 (256 IPs)
Private AZ-2:     10.0.12.0/24 (256 IPs)
```

The gap between public (/1x) and private (/11x) ranges leaves room to add subnets (e.g., database tier at 10.0.21.0/24, 10.0.22.0/24) without renumbering.

---

## IMDSv2 in UserData

All UserData scripts use the token-based Instance Metadata Service v2 (IMDSv2):
```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
```
IMDSv2 prevents SSRF attacks that could leak instance metadata. IMDSv1 (no token) is a known attack vector.

---

## SSM Agent on Amazon Linux 2023

Amazon Linux 2023 ships with the SSM agent pre-installed and enabled. The UserData for app instances explicitly ensures it is running:
```bash
systemctl enable amazon-ssm-agent
systemctl start amazon-ssm-agent
```
This is defensive — it would start on its own — but makes the intent explicit in the IaC.

---

## CloudFormation Best Practices Used

1. **SSM Parameter Store AMI** — `LatestAmiId` uses `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` to always resolve to the latest Amazon Linux 2023 AMI. No hardcoded AMI IDs.
2. **DependsOn for EIPs** — NAT EIPs depend on `InternetGatewayAttachment` to prevent race conditions.
3. **Fn::Sub for naming** — All resource names use `!Sub "${ProjectTag}-..."` for consistent, prefixed naming.
4. **Outputs with Export** — The VPC ID is exported for potential cross-stack references.
5. **Metadata UI Groups** — Parameters grouped logically in the CloudFormation console.
6. **Tags on everything** — `Name`, `Project`, `Tier`, `AZ` tags on all resources for filtering and cost allocation.