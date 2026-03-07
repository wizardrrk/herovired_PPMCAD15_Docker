
# Route 53 - Pre-Read

### What is Route 53?

**Amazon Route 53** is AWS's **DNS (Domain Name System)** service. It translates human-readable domain names (like `www.myapp.com`) into IP addresses (like `54.23.112.87`) that computers use to find each other.

The name "Route 53" comes from **port 53**, which is the standard port for DNS traffic.

### Why Does This Matter for Containers?

When you deploy containerized apps on AWS (ECS, EKS, Fargate), you need a way for users to reach them via a domain name instead of raw IP addresses or load balancer URLs.

```
Without Route 53:
─────────────────────────────────
  Users access: http://cicd-train-alb-1234567890.us-east-1.elb.amazonaws.com
  → Ugly, hard to remember, changes if you recreate the load balancer

With Route 53:
─────────────────────────────────
  Users access: https://app.mycompany.com
  → Clean, branded, doesn't change even if infra changes behind it
```

### Core Concepts

**1. Hosted Zone**
A hosted zone is a container for DNS records for a specific domain. Route 53 supports two types: public and private.

**Public Hosted Zone**
Routes traffic from the internet. Records are resolvable by anyone worldwide.
```
Public Hosted Zone: mycompany.com
  ├── A Record:     app.mycompany.com     → 54.23.112.87
  ├── CNAME Record: www.mycompany.com     → app.mycompany.com
  ├── Alias Record: api.mycompany.com     → ALB DNS name
  └── MX Record:    mycompany.com         → mail server
```

- Created automatically when you register a domain through Route 53, or manually for domains registered elsewhere.
- Route 53 assigns four name servers (an NS record set) that serve as the authoritative DNS for the zone.
- You must point your domain registrar's NS records to these Route 53 name servers for resolution to work.

**Private Hosted Zone**
Routes traffic within one or more VPCs. Records are only resolvable from associated VPCs - not from the internet.
```
Private Hosted Zone: internal.mycompany.com  (associated with vpc-0a1b2c3d)
  ├── A Record:     db.internal.mycompany.com       → 10.0.3.45
  ├── A Record:     cache.internal.mycompany.com     → 10.0.4.12
  ├── CNAME Record: api.internal.mycompany.com       → internal-alb-123456.us-east-1.elb.amazonaws.com
  └── SRV Record:   _grpc._tcp.internal.mycompany.com → 10.0.5.20:50051
```

- Must be associated with at least one VPC. Can be associated with VPCs across different accounts and regions.
- Requires `enableDnsSupport` and `enableDnsHostnames` set to `true` on the VPC.
- The domain name does not need to be registered — you can use any domain (e.g., `internal.mycompany.com`, `corp.local`, or even `mycompany.com` to override public records within the VPC).
- Useful for service discovery, internal microservice routing, and keeping database/cache endpoints off the public internet.

**Split-Horizon DNS**
You can create both a public and private hosted zone for the same domain name. VPC resources resolve against the private zone, while external users resolve against the public zone.
```
Public Zone:  mycompany.com  → app.mycompany.com → 54.23.112.87   (internet-facing)
Private Zone: mycompany.com  → app.mycompany.com → 10.0.2.30      (VPC-internal)
```

This lets internal traffic stay on private networks while external users hit public endpoints - same hostname, different answers depending on where the query originates.

**2. Common DNS Record Types**

```
Record Type │ Purpose                            │ Example
────────────┼────────────────────────────────────┼─────────────────────────
A           │ Maps domain to IPv4 address        │ app.com → 54.23.112.87
AAAA        │ Maps domain to IPv6 address        │ app.com → 2001:db8::1
CNAME       │ Maps domain to another domain name │ www.app.com → app.com
Alias       │ AWS-specific: maps to AWS resource │ app.com → ALB DNS name
MX          │ Mail server routing                │ app.com → mail.app.com
TXT         │ Text records (verification, SPF)   │ app.com → "v=spf1..."
NS          │ Name server delegation             │ app.com → ns1.aws.com
```

**3. Routing Policies**

Route 53 supports different ways to route traffic:

```
Policy              │ How It Works
────────────────────┼──────────────────────────────────────────────
Simple              │ One domain → one destination
Weighted            │ Split traffic: 80% to v1, 20% to v2
Latency-based       │ Route to the region with lowest latency
Failover            │ Primary fails → route to secondary (DR)
Geolocation         │ Route based on user's location
Multi-value answer  │ Return multiple IPs (basic load balancing)
```

### Typical Setup: Route 53 + ECS + ALB

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│  User    │ ──→ │  Route 53    │ ──→ │  ALB (Load  │ ──→ │  ECS     │
│  Browser │     │  DNS lookup  │     │  Balancer)  │     │  Tasks   │
│          │     │              │     │             │     │          │
│ app.com  │     │ Resolves to  │     │ Distributes │     │ Container│
│          │     │ ALB address  │     │ traffic     │     │ Container│
└──────────┘     └──────────────┘     └─────────────┘     └──────────┘
```