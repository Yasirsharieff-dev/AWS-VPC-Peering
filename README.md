# AWS VPC Peering Project

Connect two VPCs. Prove private connectivity.

A hands-on AWS networking lab. Two isolated VPCs are joined with a VPC peering connection so that two private EC2 servers can talk to each other over their private IPv4 addresses. A single public bastion host is the only way in: I administer both private servers through SSH ProxyJump, and the private key never leaves my laptop.

Student	: Yasir sharief Qurashi

Region	Mumbai (ap-south-1)

Completed	25 October 2026

Full report	: Yasir_Sharief_Qurashi_EWB_VPC_Peering report.pdf

---

## Architecture
<img width="2800" height="2020" alt="ewb_vpc_peering_architecture" src="https://github.com/user-attachments/assets/196bd949-96b7-4431-8798-dd3cdcf5065b" />



- **Administration path:** laptop, then bastion, then App-A or App-B (SSH ProxyJump).
- **Application traffic:** App-A and App-B talk over private IPs through the peering connection.
- VPC B has **no internet gateway**, and neither application server has a public IP.

### Address plan

| Resource | Name tag | CIDR / private IP |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public subnet | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet | `ewb-a-private` | `10.10.2.0/24` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private subnet | `ewb-b-private` | `10.20.1.0/24` |
| Bastion | `ewb-bastion` | `10.10.1.10` plus a public IPv4 |
| App-A | `ewb-app-a` | `10.10.2.10` (no public IP) |
| App-B | `ewb-app-b` | `10.20.1.10` (no public IP) |

All three instances run Amazon Linux 2023 on `t3.micro`. Other resources: internet gateway `ewb-igw-a`, route tables `ewb-rt-a-public`, `ewb-rt-a-private` and `ewb-rt-b-private`, security groups `ewb-sg-bastion`, `ewb-sg-app-a` and `ewb-sg-app-b`, peering connection `ewb-a-b-peer`, key pair `ewb-lab`.

### Resource IDs

| Resource | ID |
|---|---|
| `ewb-vpc-a` | `vpc-0a195739f08ce7d59` |
| `ewb-vpc-b` | `vpc-05166dbddd3e1b54a` |
| `ewb-a-public` | `subnet-03019e9904066ce3d` |
| `ewb-a-private` | `subnet-05482c0832db2cc93` |
| `ewb-b-private` | `subnet-091ea124a7f25b03f` |
| `ewb-igw-a` | `igw-0e4122ef5cbca2145` |
| `ewb-a-b-peer` | `pcx-0aea73d442174951b` |
| `ewb-rt-a-public` | `rtb-041f1ac148cb6e157` |
| `ewb-rt-a-private` | `rtb-00763f404aa0caec3` |
| `ewb-rt-b-private` | `rtb-0a071720960f64016` |
| `ewb-sg-bastion` | `sg-0b053bcdfa5169dc9` |
| `ewb-sg-app-a` | `sg-0c9ff9c451e186b1a` |
| `ewb-sg-app-b` | `sg-07d3c238f130a7e4d` |
| `ewb-bastion` | `i-0f87409ac8b536d0f` |
| `ewb-app-a` | `i-03be21c0e22ce10b7` |
| `ewb-app-b` | `i-02d4f04fa4986bd35` |

These resources were deleted after the lab, so the IDs no longer exist in AWS.

---

## What I built and practised

- Chose VPC and subnet CIDR ranges that do not overlap.
- Created public and private subnets, each with an explicit route table association.
- Requested, accepted and activated a peering connection, then added routes on both sides.
- Wrote narrow security group rules: single `/32` sources only, nothing open to `0.0.0.0/0` for SSH or ICMP.
- Reached private servers through a bastion with SSH ProxyJump instead of copying keys onto servers.
- Tested the design with a failed test, successful tests and a controlled failure.
- Removed every lab resource afterwards.

---

## Routing

| Route table | Destination | Target | Purpose |
|---|---|---|---|
| `ewb-rt-a-public` | `10.10.0.0/16` | local | Bastion to App-A |
| `ewb-rt-a-public` | `0.0.0.0/0` | internet gateway | Bastion internet access |
| `ewb-rt-a-public` | `10.20.0.0/16` | peering | Bastion SSH to App-B |
| `ewb-rt-a-private` | `10.10.0.0/16` | local | VPC A traffic |
| `ewb-rt-a-private` | `10.20.0.0/16` | peering | App-A to VPC B |
| `ewb-rt-b-private` | `10.20.0.0/16` | local | VPC B traffic |
| `ewb-rt-b-private` | `10.10.0.0/16` | peering | Return path to VPC A |

Evidence: [internet gateway](screenshots/02_internet_gateway_attached.png), [public route table](screenshots/03_rt-a-public_routes.png), [App-A private route table](screenshots/04_rt-a-private_routes.png), [App-B private route table](screenshots/05_rt-b-private_routes.png), [peering Active](screenshots/06_peering_active.png).

## Security group inbound rules

| Security group | Type | Source |
|---|---|---|
| `ewb-sg-bastion` | SSH / TCP 22 | my public IPv4 `/32` |
| `ewb-sg-app-a` | SSH / TCP 22 | `10.10.1.10/32` (bastion) |
| `ewb-sg-app-a` | ICMP echo request | `10.20.1.10/32` (App-B) |
| `ewb-sg-app-b` | SSH / TCP 22 | `10.10.1.10/32` (bastion) |
| `ewb-sg-app-b` | ICMP echo request | `10.10.2.10/32` (App-A) |

Outbound rules are the default allow-all, and the network ACLs are the default allow-all. Evidence: [bastion SG](screenshots/07_sg_bastion.png), [App-A SG](screenshots/08_sg_app-a.png), [App-B SG](screenshots/09_sg_app-b.png).

---

## SSH setup (ProxyJump)

Every SSH command runs on my laptop. The `.pem` key stays there.

```
Host ewb-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

I ran these from Windows with the equivalent `ssh -i ... -o ProxyCommand=...` commands. Evidence: [SSH to bastion](screenshots/10_ssh_bastion.png), [SSH to App-A](screenshots/11_ssh_app-a_via_bastion.png).

---

## Test results

| # | Test | Where / command | Expected | Result | Evidence |
|---|---|---|---|---|---|
| 01 | Baseline | App-A: `ping -c 4 10.20.1.10`, before peering|
| 02 | Forward path | App-A: `ping -c 4 10.20.1.10` | 4 replies, 0% loss |
| 03 | Reverse path | App-B: `ping -c 4 10.10.2.10` | 4 replies, 0% loss | 
| 04 | Private SSH | Laptop to App-B through the bastion | Shell on App-B | Shell opened on `ip-10-20-1-10` | [screenshot](screenshots/12_test04_ssh_app-b_via_bastion.png) |
| 05 | Isolation | EC2 details of App-A and App-B | No public IPv4 | Public IPv4 empty on both | [App-A](screenshots/13_test05_isolation_app-a_no_public_ip.png), [App-B](screenshots/14_test05_isolation_app-b_no_public_ip.png) |

### Controlled failure and recovery

To show that the security group, and not just the routing, decides whether traffic is allowed, I removed only the ICMP echo rule on `ewb-sg-app-b`, left the SSH rule alone, and pinged again.

1. With the rule removed, the ping from App-A got 0 replies and 100% packet loss: [screenshot](screenshots/15_failure_ping_app-a_icmp_removed.png).
2. I restored the identical rule and pinged again. Recovery evidence: _add screenshot_.

---

## How the traffic flows

- **Forward path (App-A to App-B):** `ewb-rt-a-private` matches `10.20.0.0/16` and hands the packet to the peering connection. `ewb-sg-app-b` accepts it because the source `10.10.2.10/32` is allowed.
- **Return path (App-B to App-A):** `ewb-rt-b-private` matches `10.10.0.0/16` and sends the reply back through the same peering connection. Security groups are stateful, so the reply needs no extra inbound rule.
- **Administration:** with ProxyJump the bastion opens its own SSH connection to the target. App-B therefore sees the bastion's private IP as the source, which is why `ewb-sg-app-b` allows SSH only from `10.10.1.10/32`. It is also why `ewb-rt-a-public` needs a route to `10.20.0.0/16`: the more specific peering route wins over the internet default route.

## Key takeaways

- Peering is only a link. Routes on both sides and security group rules decide what can use it.
- Peering is one-to-one and non-transitive, and it does not give VPC B internet access through VPC A's internet gateway.
- The CIDR ranges of the two VPCs must not overlap.
- An Active peering connection does not guarantee a successful ping.

---

## Cleanup

After saving the evidence I removed everything created for this project: the instances and their volumes, the peering routes and connection, the security groups, subnets, custom route tables, internet gateway and both VPCs. Cleanup evidence: _add screenshot_.

## Repository layout

```
.
├── README.md
├── Yasir_Sharief_Qurashi_EWB_VPC_Peering.pdf   # full project report
├── architecture.png
├── .gitignore                                  #
└── screenshots/                                
```

## Security notes

- No private keys, passwords or access tokens are stored here, and `*.pem` is git-ignored.
- SSH to the bastion was allowed only from my own public IP `/32`, and nothing exposes SSH or ICMP to `0.0.0.0/0`.
- Both application servers stayed private with no public IP.

## Author

**Yasir Sharief Qurashi**
