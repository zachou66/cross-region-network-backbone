# Cross-Region Encrypted Network Backbone (AWS)

Two AWS VPCs in different regions, joined by a hand-built WireGuard site-to-site VPN, with zero public exposure on the workloads.

## Overview

This project builds a private network backbone between two AWS regions
(`us-east-1` and `us-west-2`) without AWS Transit Gateway or VPC peering. Each
region runs its own VPC; a self-managed WireGuard gateway in each region forms an
encrypted site-to-site tunnel, so private subnets in one region can reach private
subnets in the other across the public internet. The workload instances have no
public IPs and are administered only through a single bastion using SSH agent
forwarding, so the private key never leaves my PC.

## Architecture

| Network | Region | CIDR |
| --- | --- | --- |
| vpc-east | us-east-1 | 10.10.0.0/16 |
| vpc-west | us-west-2 | 10.20.0.0/16 |
| WireGuard tunnel | — | 10.99.99.0/24 |

Each VPC has a public subnet (gateway + bastion) and a private subnet (workload).
The WireGuard gateways hold the static Elastic IPs and route each region's `/16`
to the other across the tunnel; source/destination checks are disabled so the
gateways can forward for their VPCs.

## Phases

| Phase | Focus | Status |
| --- | --- | --- |
| 0 | Account guardrails, two regions, SSH key pairs | Complete |
| 1 | VPC foundation in both regions (subnets, IGW, route tables) | Complete |
| 2 | Instances, security groups, bastion + agent forwarding | Complete |
| 3 | WireGuard tunnel between the two regional gateways | Complete |
| 4 | End-to-end proofs: cross-region routing, NAT, encryption, latency | Complete |

## Phase 0 - Account Guardrails, Two Regions, Key Pairs

Set a spend tripwire and create one SSH key pair, imported into both
regions, that never leaves my PC.

The budget pretty much just catches when I hit the budget threshold for the amount spent in services/functions of AWS. It alerts me through email. One private key is imported in both regions because AWS key pairs are only per-region. A key thats imported in US-EAST-1 doesn't exist in US-WEST-2 unless you import it to both regions. If you dont import it to one region, you won't be able to launch or access any instances in that region because the public key authentication wont work.

## Phase 1 - VPC Foundation (both regions)

Build a public+private VPC in each region with an IGW and two route
tables; the private subnets have no internet route yet.

The difference between a public and a private subnet is that a private subnet doesn't have access to the internet. It is not routable through the internet. A public subnet has access to the internet. They have public ips, access to the internet gateway, and route directly to the internet. I chose non-overlapping CIDRs per region because if I didn't, both VPCs would intertwine with each other causing some kind of logical error. It doesn't make sense to have two networks with the same ip range. The routers would get confused about which side to send the traffic to, which results in a VPN communication failure.

## Phase 2 - Instances, Security Groups, Bastion Access
Launch the gateways and workloads in both regions with least-exposure
security groups; admin path is PC -> bastion(east) -> private instances, key never leaving the PC.

The bastion host pattern gives me a single, secure jump server in the public subnet so my actual workloads can stay completely private without any public IPs attached. I used SSH agent forwarding to tunnel my local authentication right through the bastion, which is huge because it means my private key never actually leaves my personal PC. I also had to explicitly turn off the source/destination checks on both WireGuard gateways. By default, AWS drops any packets that aren't addressed directly to an EC2 instance, so if you leave this on, the gateways can't actually act as routers to forward traffic between the VPCs. Finally, I had to use hardcoded CIDRs and Elastic IPs for the cross-region security group rules because AWS doesn't let you reference a security group if it lives in a totally different region.

## Phase 3 - WireGuard Cross-Region Tunnel

The tunnel is symmetric because both gateways have static Elastic IPs that never change, so either side can bring it up and it stays up on its own — no keepalive needed (the home build needed that because a house has no fixed IP and sits behind NAT). AllowedIPs does double duty: it's both the route into the tunnel and the crypto ACL for what each peer will accept, so east lists the west VPC and west lists the east. The route tables were still required because a live tunnel only connects the two gateways — until I pointed 10.20.0.0/16 at wg-east's ENI (and the mirror on the west), the private subnets had no idea the tunnel existed.
## Phase 4 - End-to-End Proofs

For the claim that I built a cross-region encrypted backbone, I proved the routing and encryption work by showing the active handshake (phase3_wg-handshake), a successful cross-region ping (phase4_cross-region-ping), and packet captures proving the traffic is opaque UDP on the wire (phase4_tcpdump-encrypted).

For the claim about VPC routing and zero public exposure, I proved it by showing the VPC maps (phase1_*-vpc-map), the custom route tables (phase3_route-tables), the fact that the workload instances have no public IPs (phase2_instances), and that they successfully use NAT for internet access (phase4_nat_egress_east / _west).

Finally, for the hardened administration claim, I proved it by showing SSH agent forwarding in action (phase2_agent-forwarding) and using the single east-coast bastion to successfully SSH straight into the west-coast workload (phase4_bastion-to-west).

## What I Learned / What Broke

Public vs. Private IP Routing within a VPC: During Phase 3, I initially attempted to connect to the wg-east gateway from the bastion using its public Elastic IP. The connection failed because the gateway's security group was strictly locked down to permit SSH only from the bastion's internal security group. I learned that to satisfy those internal-only rules, communication between the bastion and the gateway must route through their private IP addresses, keeping the traffic securely inside the VPC.

The "Missing" Private Key (SSH Agent Forwarding): I spent time troubleshooting what looked like a missing credential when I couldn't find my private key file anywhere in the ~/.ssh directory on the bastion host. It turned out this wasn't a bug, but the exact expected behavior of SSH Agent Forwarding (ssh -A) working correctly. The authentication successfully tunneled through the bastion to the private workloads, proving that I could manage the whole architecture while keeping the private key safely isolated on my local machine.

The Silent Packet Drop (Source/Destination Checks): By default, AWS drops any packet an EC2 instance forwards if it isn't specifically addressed to that instance. This meant the WireGuard tunnel could come up, but no traffic would route through it. I had to explicitly disable the Source/Destination check on both the wg-east and wg-west gateways to allow them to act as actual routers for their respective VPCs.

WireGuard's Dual-Purpose AllowedIPs: I learned that the AllowedIPs directive in WireGuard is easily misunderstood. It acts as both the routing table entry ("send these CIDRs into the tunnel") and the cryptographic ACL ("accept these source ranges from this peer"). Getting this configured correctly in a cross-region setup meant carefully ensuring the East gateway explicitly listed the West VPC CIDR, and vice versa.

Per-Region Infrastructure Isolation: A common stumbling block when building identical infrastructure across multiple availability zones is managing regional scope. Things like SSH Key Pairs and specific AMI IDs are strictly isolated per region. I had to ensure the exact same public key was explicitly imported into both us-east-1 and us-west-2, and manually select the correct regional Ubuntu AMIs rather than trying to carry IDs across coasts.