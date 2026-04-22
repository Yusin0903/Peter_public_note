---
sidebar_position: 6
---

# NAT Gateway vs Transit Gateway

Both have "Gateway" in the name, but they serve completely different purposes.

---

## NAT Gateway — "Exit Guard"

Lets **machines in private subnets access the internet**.

```
Your EC2 (private IP, can't reach internet directly)
    │
    ▼
NAT Gateway   ← Translates private IP to public IP
    │
    ▼
Public Internet   ← Data goes out through public network
    │
    ▼
Destination (e.g., GitHub, Docker Hub, external APIs)
```

**Use cases:**
- EC2 downloading packages (apt install, npm install)
- Lambda calling external APIs
- Any private subnet resource needing internet access

**Pricing:**
- $0.045/GB (data goes through public network, more expensive)
- $0.045/hour existence fee

---

## Transit Gateway — "Internal Highway"

Connects **multiple VPCs or Regions** without going through public internet.

```
Region A (e.g., ap-southeast-1)
  └── VPC A
        │
        ▼
   Transit Gateway   ← AWS backbone network, stays within AWS
        │
        ▼
   Region B (e.g., us-east-1)
  └── VPC B
```

**Use cases:**
- Services across 10 regions need to communicate
- Connecting multiple VPCs together (hub-and-spoke architecture)
- Cross-region data push (e.g., Prometheus remote_write to central storage)

**Pricing:**
- $0.02/GB (stays within AWS, cheaper than public network)
- No NAT Gateway needed, saves $0.045/GB

---

## One-Line Summary

| | NAT Gateway | Transit Gateway |
|--|:-----------:|:---------------:|
| Purpose | private → internet | VPC ↔ VPC / Region ↔ Region |
| Network path | Public internet | AWS internal backbone |
| Cost/GB | $0.045 | $0.02 |
| Typical scenario | EC2 downloading packages | Cross-region service communication |
