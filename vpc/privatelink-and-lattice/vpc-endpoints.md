# VPC Endpoints

---

## Core Objective

**Why does this exist?** By default, traffic from an EC2 instance to an AWS service (S3, DynamoDB, SSM, etc.) travels **out of the VPC**, through an Internet Gateway or NAT Gateway, over the public internet to the AWS service's public endpoint — even though both your instance and the AWS service are "inside AWS." This means:

- Traffic is exposed to the public internet (compliance risk)
- NAT Gateway incurs per-GB data processing costs
- Latency is higher than necessary
- Fine-grained network-level access control is harder to enforce

**VPC Endpoints** solve this by creating a **private, direct route from your VPC to AWS services** entirely within the AWS network — no internet, no NAT Gateway, no IGW required. Traffic never leaves the Amazon network fabric.

> **Exam lens:** VPC Endpoints are a critical cost-optimization and security topic. Scenarios involving *"reduce NAT Gateway costs,"* *"access S3 from a private subnet without internet,"* *"enforce that S3 can only be accessed from within the VPC,"* or *"private connectivity to AWS services"* all point to VPC Endpoints.

**AWS Reference:** https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html

---

## Fundamental Concepts

### The Three Types of VPC Endpoints

| Feature | **Gateway Endpoint** | **Interface Endpoint** | **Gateway Load Balancer Endpoint** |
|---|---|---|---|
| **Mechanism** | Route table entry (prefix list) | ENI with private IP in your subnet | ENI + GWLB for traffic inspection |
| **Supported services** | **S3** and **DynamoDB only** | Most AWS services + AWS Marketplace + your own services (PrivateLink) | Third-party virtual appliances (firewalls, IDS/IPS) |
| **Cost** | ✅ **Free** | ~$0.01/hr per AZ + $0.01/GB data | ~$0.01/hr + $0.004/GB |
| **DNS resolution** | Uses route table; no DNS change | Resolves service DNS to private IP (requires Private DNS enabled) | N/A (transparent proxy) |
| **Security Group** | ❌ Not applicable | ✅ Attach SG to the endpoint ENI | ✅ |
| **Endpoint Policy** | ✅ Yes | ✅ Yes | ❌ |
| **Works across regions** | ❌ Same region only | ❌ Same region only | ❌ Same region only |
| **PrivateLink-based** | ❌ | ✅ | ✅ |

---

### Key Terminology

- **ENI (Elastic Network Interface):** For Interface Endpoints, AWS provisions a real network interface inside your subnet with a private IP. All service traffic is routed to this ENI.
- **Private DNS:** When enabled on an Interface Endpoint, AWS overrides the public DNS for the service (e.g., `s3.amazonaws.com`) to resolve to the endpoint's private IP — **transparent to your application**, no code changes required.
- **Endpoint Policy:** A resource-based IAM policy attached directly to the endpoint that restricts which principals and actions can use it. Acts as an additional access control layer on top of IAM policies.
- **AWS PrivateLink:** The underlying technology powering Interface Endpoints. It uses Network Load Balancers on the service side and ENIs on the consumer side to route traffic privately. Also enables you to expose **your own services** to other VPCs or AWS accounts privately.
- **Prefix List:** For Gateway Endpoints, AWS maintains a managed prefix list (a group of IP CIDRs representing the service). Your route table entry points to the gateway endpoint using this prefix list as the destination.
- **VPC Endpoint Service:** The provider-side construct when using PrivateLink to expose your own application. You create an NLB, attach it to an Endpoint Service, and consumers create Interface Endpoints pointing to it.

---

### Gateway vs Interface — When to Use Which

```
Is the target service S3 or DynamoDB?
  └─ YES → Use Gateway Endpoint (it's free, simpler, no DNS complexity)
  └─ NO  → Use Interface Endpoint (covers everything else: SSM, ECR, KMS, SNS, SQS, etc.)

Do you need to expose YOUR OWN service privately to other VPCs/accounts?
  └─ YES → Use PrivateLink (Interface Endpoint + VPC Endpoint Service + NLB)
```

---

## Procedure or Logic

### Gateway Endpoint — How It Works

```
1. Create a Gateway Endpoint for S3 or DynamoDB in your VPC
   └─ Select which route tables to associate (typically: all private subnet route tables)

2. AWS automatically adds a route entry to each associated route table:
   └─ Destination: pl-xxxxxxxx (managed prefix list for S3/DynamoDB CIDRs)
   └─ Target: vpce-xxxxxxxxx (your gateway endpoint ID)

3. When an EC2 instance in a private subnet calls S3:
   └─ DNS resolves s3.amazonaws.com → public S3 IPs (unchanged)
   └─ Route table intercepts traffic matching the prefix list → routes to endpoint
   └─ Traffic stays inside the AWS network; never hits the internet

4. Endpoint Policy controls which S3 buckets/DynamoDB tables can be accessed
```

**Create via CLI:**
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-11111111 rtb-22222222 \
  --vpc-endpoint-type Gateway
```

---

### Interface Endpoint — How It Works

```
1. Create an Interface Endpoint for the target service (e.g., SSM, ECR, KMS)
   └─ Select the VPC, subnets (one per AZ for HA), and a Security Group
   └─ Enable Private DNS (recommended — makes it transparent to apps)

2. AWS provisions an ENI in each selected subnet with a private IP
   └─ The ENI gets both a regional DNS name and a zonal DNS name

3. Private DNS (if enabled) overrides the public service DNS:
   └─ ssm.us-east-1.amazonaws.com → resolves to the ENI's private IP
   └─ No application code changes needed

4. Outbound traffic from EC2 to the service hits the ENI → stays in AWS network
   └─ Security Group on the endpoint controls who can reach it (port 443 from instance SG)

5. Endpoint Policy further restricts which actions are allowed through the endpoint
```

**Create via CLI:**
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-aaaa subnet-bbbb \
  --security-group-ids sg-endpoint \
  --private-dns-enabled
```

---

### PrivateLink — Exposing Your Own Service

```
1. Deploy your application behind a Network Load Balancer (NLB) in your VPC (provider)

2. Create a VPC Endpoint Service pointing to the NLB:
   └─ Optionally require manual acceptance of connection requests

3. Share the Endpoint Service name with consumers (other VPCs or AWS accounts)

4. Consumers create an Interface Endpoint pointing to your Endpoint Service
   └─ Traffic flows: Consumer ENI → PrivateLink → Your NLB → Your app
   └─ IPs never overlap; no VPC peering or route management needed

5. Consumer's app calls your service using the endpoint's DNS name (or their own Route 53 alias)
```

**AWS Reference (PrivateLink):** https://docs.aws.amazon.com/vpc/latest/privatelink/create-endpoint-service.html

---

## Practical Application / Examples

### Example 1 — Gateway Endpoint: Eliminate NAT Gateway Cost for S3
A data engineering team runs EMR and Glue jobs in private subnets that process terabytes of data stored in S3. They're paying $800/month in NAT Gateway data processing fees for S3 traffic.

**Solution:** Create a Gateway Endpoint for S3 and associate it with the private subnet route tables.

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-prod \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private-1 rtb-private-2 \
  --vpc-endpoint-type Gateway
```

**Result:** S3 traffic is routed through the endpoint. NAT Gateway data processing cost for S3 drops to **$0**. The endpoint itself is free. Security improves: S3 traffic no longer traverses the public internet.

---

### Example 2 — Interface Endpoint: SSM Session Manager in a Fully Private Subnet
A company runs instances in a private subnet with no internet access. SSM Agent cannot reach `ssm.amazonaws.com` over the internet.

**Solution:** Create three Interface Endpoints with Private DNS enabled:

```bash
for SERVICE in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-private \
    --service-name com.amazonaws.us-east-1.${SERVICE} \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-private-a subnet-private-b \
    --security-group-ids sg-endpoints \
    --private-dns-enabled
done
```

**Result:** SSM Agent resolves `ssm.us-east-1.amazonaws.com` to the private ENI IP. Session Manager works with zero internet access. Compliance requirement met.

---

### Example 3 — PrivateLink: SaaS Provider Exposing API to Enterprise Customers
A SaaS company needs to expose their REST API to 20 enterprise customers, each in their own AWS account, without VPC peering, IP overlap issues, or public internet exposure.

**Solution:**
1. SaaS provider deploys API behind an NLB in their VPC.
2. Creates a VPC Endpoint Service with `acceptance_required = true`.
3. Whitelists customer AWS account IDs.
4. Each customer creates an Interface Endpoint in their VPC pointing to the Endpoint Service.
5. Customer apps call `vpce-xxxxx.vpce-svc-yyyyy.us-east-1.vpce.amazonaws.com` (or a Route 53 alias like `api.saasprovider.com` → endpoint DNS).

**Result:** Traffic flows privately, SaaS provider's VPC CIDR is never exposed, no IP overlap concerns, each customer connection is independently controllable.

---

### Example 4 — Endpoint Policy: Restrict S3 Access to Specific Bucket (Data Exfiltration Prevention)
A compliance team requires that EC2 instances in the production VPC can **only** access the company's own S3 buckets — never public S3 buckets.

**Endpoint Policy attached to the S3 Gateway Endpoint:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-prod-bucket",
        "arn:aws:s3:::company-prod-bucket/*"
      ]
    }
  ]
}
```

**Complementary S3 Bucket Policy** (enforce access only from the VPC endpoint):
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::company-prod-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "aws:sourceVpce": "vpce-xxxxxxxxx"
    }
  }
}
```

**Result:** Two-sided lock — the endpoint only allows access to the approved bucket, and the bucket only accepts requests arriving through the endpoint. Data exfiltration to external S3 buckets is blocked at the network level.

---

## Critical Considerations

- **Gateway Endpoints are free — always prefer them for S3 and DynamoDB:** There is no reason to use an Interface Endpoint for S3 or DynamoDB in most cases. The Gateway Endpoint is free, has no ENI to manage, and requires only a route table entry.

- **Interface Endpoints cost money per AZ:** Each Interface Endpoint deployed in a subnet costs ~$0.01/hr (~$7.20/month) **per AZ**. In a 3-AZ setup with 10 endpoints (common for full SSM + ECR + KMS + STS coverage), that's ~$216/month. Plan endpoint deployment carefully.

- **Private DNS requires `enableDnsHostnames` and `enableDnsSupport` on the VPC:** If either attribute is disabled, Private DNS on Interface Endpoints will silently fail. This is a frequent misconfiguration root cause.

- **Endpoint Policies are additive with IAM — the most restrictive wins:** An endpoint policy does not replace IAM. Both must allow the action for it to succeed. An overly restrictive endpoint policy can silently block legitimate traffic even when IAM allows it.

- **Gateway Endpoints do not use DNS — Interface Endpoints do:** A Gateway Endpoint works via route table manipulation. DNS for `s3.amazonaws.com` still resolves to public IPs — but the route table intercepts the traffic before it leaves the VPC. This confuses practitioners who expect a DNS change.

- **Interface Endpoints are AZ-scoped:** Each ENI lives in a specific AZ. For high availability, deploy one endpoint per AZ. Traffic to an endpoint in a different AZ crosses AZ boundaries (incurring cross-AZ data transfer cost and latency).

- **VPC Endpoint Services (PrivateLink) are TCP-only:** PrivateLink does not support UDP. If your service requires UDP, PrivateLink is not applicable.

- **Cross-region is not supported:** VPC Endpoints only work within the same AWS Region. For cross-region private connectivity, use **AWS Transit Gateway** + inter-region peering or **AWS Global Accelerator**.

- **S3 Gateway Endpoint is incompatible with S3 Transfer Acceleration:** The Transfer Acceleration endpoint (`s3-accelerate.amazonaws.com`) bypasses Gateway Endpoint routing entirely. The two features cannot be combined.

- **Security Group on Interface Endpoints:** The SG attached to the endpoint ENI must allow **inbound HTTPS (443)** from the instances that will use the endpoint. Forgetting this is a common reason Interface Endpoints appear to be configured correctly but fail at runtime.

---

## Critical Synthesis Note

> **High-level insight:** VPC Endpoints represent AWS's systematic solution to the **network gravity problem** of cloud services: as workloads scale, the cost and risk of routing internal service traffic through public infrastructure becomes prohibitive. The architectural split between Gateway Endpoints (free, route-based, S3/DynamoDB only) and Interface Endpoints (paid, DNS-based, universal) reflects a deliberate tradeoff — AWS absorbs the cost of the two highest-volume services in the Gateway model because routing S3/DynamoDB traffic through PrivateLink ENIs at AWS global scale would be economically unsustainable. Understanding this split explains why the two types exist rather than one unified model.

> **Knowledge gap / further research for SAA-C03:** The exam heavily tests the **three-way interaction between Endpoint Policies, IAM Policies, and S3 Bucket Policies** in data exfiltration prevention scenarios. Master the pattern: Gateway Endpoint Policy (restricts outbound to approved buckets) + S3 Bucket Policy with `aws:sourceVpce` condition (restricts inbound to traffic from the endpoint). Also study how **VPC Endpoints differ from VPC Peering** — a common distractor pair: Peering connects two VPCs (bidirectional, routed), while Endpoints connect a VPC to a *service* (unidirectional, no routing table sprawl).

> **Cross-disciplinary connection:** AWS PrivateLink's provider/consumer model is architecturally identical to the **Service Mesh** sidecar pattern in microservices (Istio, Linkerd) — both solve service-to-service communication across trust boundaries without exposing network topology. The key distinction: PrivateLink operates at **L4 (TCP)**, while service meshes operate at **L7 (HTTP)**. **AWS VPC Lattice** (this folder's sibling service) bridges exactly this gap — bringing L7 routing, mutual TLS auth, and observability to the same private connectivity model that PrivateLink pioneered at L4.

---

**Primary AWS Documentation:** https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html

**Gateway Endpoints for S3:** https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html

**Gateway Endpoints for DynamoDB:** https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-dynamodb.html

**Interface Endpoints (PrivateLink):** https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html

**VPC Endpoint Services (expose your own service):** https://docs.aws.amazon.com/vpc/latest/privatelink/create-endpoint-service.html

**Endpoint Policies:** https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-access.html

**AWS PrivateLink pricing:** https://aws.amazon.com/privatelink/pricing/
