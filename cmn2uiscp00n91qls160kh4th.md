---
title: "How BGP silently broke my Site-to-Site VPN (GCP ↔ AWS) and how I fixed it"
seoTitle: "BGP Broke My GCP–AWS VPN: Debugging Guide"
seoDescription: "Debugging a GCP-AWS VPN failure caused by BGP link-local IP mismatch, route issues, and firewall gaps in a real production setup."
datePublished: 2026-03-23T07:10:01.227Z
cuid: cmn2uiscp00n91qls160kh4th
slug: bgp-site-to-site-vpn-gcp-aws-debugging
cover: https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/6ed88715-306e-4250-9f1e-961da23389ba.png
ogImage: https://cdn.hashnode.com/uploads/og-images/66f590c9120870cbbe3a72f4/b12de9b3-faf4-4f2a-89ab-719707242d8b.png
tags: aws, devops, networking, terraform, gcp, bgp, cloudengineering, border-gateway-protocol, learningtogether

---

## DevOps Production Engineering Series — Blog 2

### My VPN Was "UP"... But Nothing Worked

No errors. No alerts. No traffic.

Just silence — and a VPN dashboard showing everything was healthy.

That's the moment I learned that in multi-cloud networking, **"UP" means almost nothing**. This post is about the real control plane behind Site-to-Site VPN, the silent failure it caused in production, and exactly how I debugged and fixed it.

* * *

## What Is BGP, and Why Should You Care

Border Gateway Protocol (BGP) is the routing protocol that tells two connected networks **which routes exist and how to reach them**. Without BGP, even a fully established VPN tunnel is just an encrypted pipe going nowhere.

Think of it this way:

> IPSec builds the tunnel. BGP fills it with directions.

In a Site-to-Site VPN between GCP and AWS, BGP runs **inside** the tunnel over link-local addresses. If BGP does not form correctly, no routes are exchanged, and no traffic flows — even though the tunnel itself reports as UP.

* * *

## The Architecture: GCP to AWS, Connected Over IPSec + BGP

Before the bug, here is the full picture of what I was building:  

![](https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/d33f7ec7-46fb-40f4-8317-e0f55ff30791.png align="center")

Two VPN tunnels for redundancy. BGP peers established inside each tunnel over link-local addresses (`169.254.x.x`). Routes exchanged dynamically so each cloud knows how to reach the other's subnets.

That was the plan. Here is what actually happened.

* * *

## The Setup (Terraform + gcloud)

### AWS Side — VPN Gateway, Customer Gateway, VPN Connection

```yaml
resource "aws_vpn_gateway" "vgw" {
  vpc_id          = aws_vpc.main.id
  amazon_side_asn = 64512
}

resource "aws_customer_gateway" "gcp" {
  bgp_asn    = 65000
  ip_address = var.gcp_vpn_public_ip   # GCP HA VPN Gateway public IP
  type       = "ipsec.1"
}

resource "aws_vpn_connection" "gcp_tunnel" {
  vpn_gateway_id      = aws_vpn_gateway.vgw.id
  customer_gateway_id = aws_customer_gateway.gcp.id
  type                = "ipsec.1"
  static_routes_only  = false          # BGP mode, not static
}
```

After `terraform apply`, AWS created two tunnels and auto-generated:

*   Tunnel public IPs: `13.126.210.123` and `52.66.171.59`
    
*   Pre-shared keys for IPSec authentication
    
*   **Tunnel inside CIDRs** for BGP: `169.254.16.72/30` and `169.254.17.0/30`
    

### GCP Side — HA VPN Gateway + Cloud Router

```yaml
# Create the HA VPN Gateway
gcloud compute vpn-gateways create credresolve-vpn-gw \
  --network=credresolve-vpc \
  --region=us-east1

# Create a Cloud Router for BGP
gcloud compute routers create credresolve-router \
  --network=credresolve-vpc \
  --asn=65000 \
  --region=us-east1

# Create External VPN Gateway representing AWS
gcloud compute external-vpn-gateways create aws-vpn-gw \
  --interfaces 0=13.126.210.123,1=52.66.171.59

# Create IPSec tunnels
gcloud compute vpn-tunnels create tunnel-to-aws-1 \
  --peer-external-gateway=aws-vpn-gw \
  --peer-external-gateway-interface=0 \
  --vpn-gateway=credresolve-vpn-gw \
  --interface=0 \
  --ike-version=2 \
  --shared-secret=<AWS_PRESHARED_KEY_1> \
  --router=credresolve-router \
  --region=us-east1
```

* * *

## The Failure: Everything "UP", Nothing Working

After all configuration was applied:

```plaintext
VPN Tunnel Status    : ESTABLISHED
BGP Session Status   : Active
Traffic Flow         : 0 bytes
DB Connection        : Timeout
```

The tunnel was up. BGP said it was active. But no routes were exchanged, and a simple `nc` from a GCP VM to AWS RDS on port 5432 timed out every time.

This is the worst category of infrastructure bug: **no errors, just silence**.

* * *

## Root Cause: BGP Link-Local IP Mismatch

BGP sessions inside VPN tunnels use **link-local addresses** (`169.254.x.x`) as the peering IPs. Both sides must use the exact IPs that the other side expects.

Here is what went wrong:  

![](https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/47557d19-4ff9-4e03-b5cd-f146adf77aa8.png align="center")

GCP auto-assigned a random link-local IP (`169.254.109.57`) when the BGP peer was created.  
AWS had already assigned a fixed IP (`169.254.16.73`) from the tunnel inside CIDR it provided.

These did not match. BGP peering attempted, failed silently, and the tunnel stayed up looking healthy.

* * *

## The Fix: Delete, Query, Recreate with Correct IPs

### Step 1 — Delete the incorrect BGP peers on GCP

```bash
gcloud compute routers remove-bgp-peer credresolve-router \
  --peer-name=bgp-peer-tunnel-1 \
  --region=us-east1
```

### Step 2 — Query AWS for the exact inside CIDRs

```shell
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[0].Options.TunnelOptions[*]      {IP:TunnelInsideCidr}'\
  --output table
```

Output:

```plaintext
--------------------------------
|   DescribeVpnConnections     |
+------------------------------+
|  169.254.16.72/30            |  <-- Tunnel 1 CIDR, AWS side = .73, GCP side = .74
|  169.254.17.0/30             |  <-- Tunnel 2 CIDR
+------------------------------+
```

### Step 3 — Recreate BGP interfaces with the correct IPs

```bash
# Add router interface with the correct IP from AWS's CIDR
gcloud compute routers add-interface credresolve-router \
  --interface-name=if-tunnel-1 \
  --vpn-tunnel=tunnel-to-aws-1 \
  --ip-address=169.254.16.74 \         # GCP's IP inside the /30
  --mask-length=30 \
  --region=us-east1

# Add BGP peer pointing to AWS's IP
gcloud compute routers add-bgp-peer credresolve-router \
  --peer-name=bgp-peer-tunnel-1 \
  --interface=if-tunnel-1 \
  --peer-ip-address=169.254.16.73 \    # AWS's IP inside the /30
  --peer-asn=64512 \
  --region=us-east1
```

The `/30` subnet gives 4 addresses. AWS takes `.73`, GCP takes `.74`. That is it — nothing more complex than that, but the mismatch breaks everything.

* * *

## The Second Hidden Issue: Firewall Rules

After fixing BGP, routes were exchanged. But traffic still did not reach AWS RDS.

The GCP firewall rules only allowed internal GCP CIDRs as source ranges:

```plaintext
credresolve-allow-postgres  →  source: 10.0.0.0/8   (GCP internal only)
credresolve-allow-internal  →  source: 10.0.0.0/8   (GCP internal only)
```

Traffic coming **from AWS** arrives with source IP from `172.31.0.0/16`. GCP was dropping it at the firewall — silently.

### Fix: Add the AWS VPC CIDR to both rules

```bash
gcloud compute firewall-rules update credresolve-allow-postgres \
  --source-ranges=10.0.0.0/8,172.31.0.0/16

gcloud compute firewall-rules update credresolve-allow-internal \
  --source-ranges=10.0.0.0/8,172.31.0.0/16
```

* * *

## The Full BGP Peering Flow (After Fix)  

![](https://cdn.hashnode.com/uploads/covers/66f590c9120870cbbe3a72f4/22d4e18e-aa44-4a89-a700-3063c53d6b09.png align="center")

* * *

## Verification: How to Confirm BGP Is Actually Working

Do not trust the dashboard. Run these.

### On GCP — Check BGP peer status and learned routes

```bash
# Check BGP peer status
gcloud compute routers get-status credresolve-router \
  --region=us-east1 \
  --format="json(result.bgpPeerStatus)"

# Expected output
{
  "name": "bgp-peer-tunnel-1",
  "ipAddress": "169.254.16.74",
  "peerIpAddress": "169.254.16.73",
  "status": "UP",
  "numLearnedRoutes": 1,
  "linkedVpnTunnel": "tunnel-to-aws-1"
}
```

### On AWS — Verify VPN connection status

```bash
aws ec2 describe-vpn-connections \
  --query 'VpnConnections[*].VgwTelemetry[*].{IP:OutsideIpAddress,Status:Status,BGP:AcceptedRouteCount}' \
  --output table

# Expected output
-----------------------------------------------
|      VPN Telemetry                          |
+-------------------+----------+--------------+
|  IP               |  Status  |  BGP Routes  |
+-------------------+----------+--------------+
|  13.126.210.123   |  UP      |  2           |
|  52.66.171.59     |  UP      |  2           |
+-------------------+----------+--------------+
```

### End-to-end connectivity test from GCP VM

```bash
# Test TCP connectivity to AWS RDS on port 5432
nc -zv <aws-rds-endpoint> 5432

# Expected
Connection to <aws-rds-endpoint> 5432 port [tcp/postgresql] succeeded!
```

* * *

## Troubleshooting Checklist

Use this when your VPN is UP but traffic is not flowing.

```plaintext
LAYER 1 — TUNNEL
  [ ] IPSec tunnel is in ESTABLISHED state on both sides
  [ ] Pre-shared keys match exactly on both sides
  [ ] IKE version matches (IKEv2 on both sides)

LAYER 2 — BGP PEERING
  [ ] Link-local peer IPs match the AWS-assigned inside CIDRs (/30 subnets)
  [ ] ASNs are correct (GCP: 65000, AWS: 64512)
  [ ] BGP peer status is UP (not just "Active" or "Established")
  [ ] numLearnedRoutes > 0 on GCP side
  [ ] AcceptedRouteCount > 0 on AWS VGW telemetry

LAYER 3 — ROUTING
  [ ] AWS route table has VPN Gateway propagation enabled
  [ ] GCP VPC has learned routes from Cloud Router
  [ ] Routes are actually in the routing table, not just advertised

LAYER 4 — FIREWALL
  [ ] GCP firewall allows source range 172.31.0.0/16 (AWS VPC)
  [ ] AWS Security Groups allow GCP source IPs
  [ ] No NACLs blocking traffic on the AWS side
  [ ] PostgreSQL is listening on 0.0.0.0 (not 127.0.0.1)
  [ ] pg_hba.conf allows the AWS CIDR range

LAYER 5 — APPLICATION
  [ ] Database accepts connections from VPN source IP
  [ ] Correct port is open end-to-end
  [ ] Test with netcat before testing with the application
```

* * *

## What I Learned

**VPN "UP" is a Layer 3 status. BGP lives above it.**

The tunnel being established only means IPSec handshake succeeded and encryption is working. It says nothing about whether routes are being exchanged, which is entirely BGP's job.

The specific lessons from this incident:

*   BGP link-local IPs must match the AWS-assigned inside CIDRs exactly. GCP will auto-assign random ones if you do not specify.
    
*   Always verify `numLearnedRoutes`, not just peer status. A peer can be "Active" without having exchanged a single route.
    
*   Firewall rules are a second independent failure layer. Even after BGP is fixed, traffic can still be dropped silently.
    
*   Run `nc` (netcat) end-to-end before debugging at the application layer. It removes all application-level variables from the equation.
    

* * *

## Closing

This was one of those infrastructure problems where the system reports success at every individual component, and the failure only becomes visible when you look at the system as a whole.

If you are building multi-cloud connectivity, do not stop at tunnel status. Verify BGP. Verify routes. Verify firewall rules. And then verify the actual packet reaching the destination.

More real-world DevOps bugs, system design, and production fixes in the next post.

* * *

*DevOps Production Engineering Series*  
*Blog 1: Preventing concurrent Jenkins deployments using deployment locks*  
*Blog 2: BGP mismatch in Site-to-Site VPN — silent failure, real impact*

`#DevOps` `#AWS` `#GCP` `#BGP` `#Networking` `#CloudEngineering` `#Terraform` `#LearningTogether`