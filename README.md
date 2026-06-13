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

<!-- Write-up goes here. -->

## Phase 1 - VPC Foundation (both regions)

<!-- Write-up goes here. -->

## Phase 2 - Instances, Security Groups, Bastion Access

<!-- Write-up goes here. -->

## Phase 3 - WireGuard Cross-Region Tunnel

<!-- Write-up goes here. -->

## Phase 4 - End-to-End Proofs

<!-- Write-up goes here. -->

## What I Learned / What Broke

<!-- The honest section. Real failures and their diagnoses go here. -->
