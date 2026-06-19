# Cross-Region Encrypted Network Backbone (AWS)

<!-- One-line description of the lab. -->

## Overview

<!-- What this lab proves and why: two AWS VPCs in different regions, joined by a
     hand-built WireGuard site-to-site VPN, with zero public exposure on the workloads. -->

## Architecture

<!-- Diagram + address plan go here.
     vpc-east  us-east-1  10.10.0.0/16
     vpc-west  us-west-2  10.20.0.0/16
     tunnel    10.99.99.0/24 -->

## Phases

| Phase | Focus | Status |
| --- | --- | --- |
| 0 | Account guardrails, two regions, SSH key pairs | Not started |
| 1 | VPC foundation in both regions (subnets, IGW, route tables) | Not started |
| 2 | Instances, security groups, bastion + agent forwarding | Not started |
| 3 | WireGuard tunnel between the two regional gateways | Not started |
| 4 | End-to-end proofs: cross-region routing, NAT, encryption, latency | Not started |

## Phase 0 - Account Guardrails, Two Regions, Key Pairs

Set a spend tripwire and create one SSH key pair, imported into both
regions, that never leaves my PC.

The budget pretty much just catches when I hit the budget threshold for the amount spent in services/functions of AWS. It alerts me through email. One private key is imported in both regions because AWS key pairs are only per-region. A key thats imported in US-EAST-1 doesn't exist in US-WEST-2 unless you import it to both regions. If you dont import it to one region, you won't be able to launch or access any instances in that region because the public key authentication wont work.

## Phase 1 - VPC Foundation (both regions)

Build a public+private VPC in each region with an IGW and two route
tables; the private subnets have no internet route yet.

The difference between a public and a private subnet is that a private subnet doesn't have access to the internet. It is not routable through the internet. A public subnet has access to the internet. They have public ips, access to the internet gateway, and route directly to the internet. I chose non-overlapping CIDRs per region because if I didn't, both VPCs would intertwine with each other causing some kind of logical error. It doesn't make sense to have two networks with the same ip range. The routers would get confused about which side to send the traffic to, which results in a VPN communication failure


## Phase 2 - Instances, Security Groups, Bastion Access
Launch the gateways and workloads in both regions with least-exposure
security groups; admin path is PC -> bastion(east) -> private instances, key never leaving the PC.



## Phase 3 - WireGuard Cross-Region Tunnel

Troubleshooting Note
I tried connecting to the wg-east instance from the bastion using its public Elastic IP, but it failed because the instance's security group strictly permits internal traffic. The solution is to always use the instance's private IP address when communicating from within the same VPC.

wg-east public key for wireguard: qspR2qck14MoDjTD5nzBp8sHD1ExYib6E4F4czIv53M=
wg-west public key for wireguard: l/E5NGFq1lpXvs0jxHrNGeWoOleiSezS4dQwfpTxTG4=
## Phase 4 - End-to-End Proofs

<!-- Write-up goes here. -->

## What I Learned / What Broke

<!-- The honest section. Real failures and their diagnoses go here. -->
