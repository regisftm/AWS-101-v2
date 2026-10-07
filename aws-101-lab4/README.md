# AWS-101 Lab 4: Site-to-Site VPN Configuration

## Overview

> **Scenario:** The application server is live, but it depends on databases and identity services that are still at Redwood's HQ. Connect AWS to HQ privately, without adding a new security stack: FortiGate at both ends, with the same policies and logs.

Build an IPsec tunnel between your AWS FortiGate and your HQ (on-premises) FortiGate. When you finish, `10.100.0.0/16` (AWS) and `192.168.0.0/22` (HQ) can reach each other.

**Prerequisites:** Labs 1–3 completed, plus the HQ details from [What You Need](/README.md#what-you-need).

**Your HQ environment** (hosted by your instructor, one per student):

| Component | Address |
| --- | --- |
| HQ FortiGate public IP | `<on-prem-public-ip>` |
| HQ FortiGate `port2` (LAN) | `192.168.2.4` |
| HQ Windows VM | `192.168.2.10` (RDP via `<on-prem-public-ip>:9833`) |

![REFERENCE ARCHITECTURE](images/reference-architecture-final.png)

> [!IMPORTANT]
> **NAT Traversal must be enabled.** AWS maps the Elastic IP to `port1`'s private address, so IPsec must run over UDP/4500.

A FortiGate-to-FortiGate tunnel avoids the hourly charges of AWS Site-to-Site VPN and Transit Gateway, and keeps one policy model and one set of logs at both ends. You still pay for the EC2 instance and for data transferred out of AWS.

<details>
<summary><b>NAT Traversal (NAT-T) explained</b></summary>

**The problem:** IPsec encrypts traffic with ESP (IP protocol 50), which has no port numbers. NAT devices rely on ports to track flows, so plain ESP often breaks across NAT.

**NAT detection:** during IKE Phase 1, each peer sends a hash of its own IP and port. In this lab, the on-prem peer sends to the Elastic IP, but the AWS FortiGate sees its private `10.100.1.x`. The hashes don't match, so NAT is detected.

**The fix (RFC 3948):** both peers wrap ESP in UDP/4500, and IKE also moves from UDP/500 to UDP/4500. The UDP header gives NAT devices ports to track, and the ESP payload is unchanged.

```text
Without NAT-T: [ IP | ESP | encrypted payload ]          ← no ports
With NAT-T:    [ IP | UDP 4500 | ESP | encrypted payload ] ← NAT can track it
```

**Why it's mandatory here:** the IGW performs 1:1 NAT between the Elastic IP and `port1`. The security group also allows only UDP/500 and UDP/4500, not raw ESP.

**Keepalives vs. DPD:** NAT-T keepalives (every 10 s) keep idle UDP state alive along the path. Dead Peer Detection is a separate IKE check that detects a dead peer and clears stale SAs.

| Item | Detail |
| --- | --- |
| ESP | IP protocol 50, no ports |
| IKE | UDP/500, then UDP/4500 once NAT is detected |
| NAT-T | ESP wrapped in UDP/4500 |
| AWS IGW | 1:1 NAT between the EIP and `port1`, which is why NAT-T is mandatory |
</details>

---

## VPN Parameters

| Parameter | On-Premises FortiGate | AWS FortiGate |
| --- | --- | --- |
| Tunnel name | `to_aws` | `to_on_prem` |
| Remote peer IP | `<FGT-EIP>` | `<on-prem-public-ip>` |
| Remote subnets | `10.100.0.0/16` | `192.168.0.0/22` |
| Local subnets | `192.168.0.0/22` | `10.100.0.0/16` |
| WAN / LAN interface | `port1` / `port2` | `port1` / `port2` |
| Pre-shared key | `RedwoodIndustries2026!` | `RedwoodIndustries2026!` |
| IKE / NAT-T / Keepalive | IKEv2 / Enabled / 10 s | IKEv2 / Enabled / 10 s |

---

## Step 1: Allow IPsec to the AWS FortiGate

1. Open **EC2 → Security Groups** and select `redwood-aws101-lab-fgt-sg`.

   ![SECURITY GROUPS](images/step1.1.png)

2. Click **Edit inbound rules** and add:

   | Type | Port | Source | Description |
   | --- | --- | --- | --- |
   | Custom UDP | 500 | `<on-prem-public-ip>/32` | IKE |
   | Custom UDP | 4500 | `<on-prem-public-ip>/32` | IPsec NAT-T |

   ![SECURITY GROUPS RULES](images/step1.2.png)

Restrict the source to the on-prem `/32`. Internet-exposed IKE endpoints get probed within minutes. No other AWS change is needed, because security groups allow all outbound traffic by default.

---

## Step 2: Configure the On-Premises FortiGate (`to_aws`)

1. Log in to your HQ FortiGate at `https://<on-prem-public-ip>` with the credentials from your instructor.

2. Open **VPN → VPN Wizard**. Set **Tunnel Name** to `to_aws` and **Template** to `Site to Site`, then click **Begin**.

   ![VPN WIZ](images/step3.1.png)

3. **VPN Tunnel:**

   | Parameter | Value |
   | --- | --- |
   | Authentication method | `Pre-shared Key` |
   | Pre-shared Key | `RedwoodIndustries2026!` |
   | IKE | `Version 2` |
   | Transport | `UDP` |
   | NAT Traversal | `Enable` |
   | Keepalive frequency | `10` |

   ![VPN TUNNEL](images/step3.2.png)

   > In production, use a random PSK of at least 20 characters, or certificate authentication.

4. **Remote Site:**

   | Parameter | Value |
   | --- | --- |
   | Remote site device type | Fortinet |
   | Remote site device | `Accessible and static` |
   | IP/FQDN | `<FGT-EIP>` |
   | Route this device's internet traffic through the remote site | `Off` |
   | Remote site subnets | `10.100.0.0/16` |

   ![REMOTE SITE](images/step3.3.png)

5. **Local Site:**

   | Parameter | Value |
   | --- | --- |
   | Outgoing interface | `port1` |
   | Create and add interface to zone | `Off` |
   | Local site | `port2` |
   | Local subnets | `192.168.0.0/22` |
   | Allow remote site's internet traffic through this device | `Off` |

   ![LOCAL SITE](images/step3.4.png)

6. Review and click **Submit**. The tunnel stays **Inactive** until the AWS side is configured.

   ![REVIEW](images/step3.5.png)

<details>
<summary><b>What the wizard creates</b></summary>

| Object | On-prem | AWS |
| --- | --- | --- |
| Tunnel interface (Phase 1 + 2) | `to_aws` | `to_on_prem` |
| Static route | `10.100.0.0/16 → to_aws` | `192.168.0.0/22 → to_on_prem` |
| Outbound policy | `port2 → to_aws` | `port2 → to_on_prem` |
| Inbound policy | `to_aws → port2` | `to_on_prem → port2` |
| Address objects | local and remote subnets | local and remote subnets |

In production, review the auto-created policies and tighten the sources and destinations if you don't want any-to-any across the tunnel.
</details>

---

## Step 3: Configure the AWS FortiGate (`to_on_prem`)

On the AWS FortiGate, run the same wizard with the **AWS FortiGate** column from [VPN Parameters](#vpn-parameters):

- Tunnel Name: `to_on_prem`
- VPN Tunnel settings: same as Step 2
- Remote Site IP: `<on-prem-public-ip>`, remote subnets `192.168.0.0/22`
- Local Site: `port1` / `port2`, local subnets `10.100.0.0/16`

![REVIEW AWS](images/step4.5.png)

After you click **Submit**, the tunnel should come up within about 30 seconds.

![VERIFICATION](images/step4.7.png)

---

## Step 4: Verify the Tunnel

On both FortiGates, open **Dashboard → Network → IPsec** and confirm the tunnel is **Up**.
<!-- TODO: verify the IPsec monitor GUI path for the FortiOS version in use -->

![TO_ON_PREM](images/step5.1.png)
![TO_AWS](images/step5.2.png)

Optional CLI check:

```text
diagnose vpn ike gateway list name to_on_prem
diagnose vpn tunnel list name to_on_prem
```

---

## Step 5: Test Cross-Site Connectivity

**HQ → AWS:** RDP to the HQ Windows VM at `<on-prem-public-ip>:9833` with the credentials from your instructor. Then run in PowerShell:

```powershell
Test-NetConnection -ComputerName 10.100.2.10 -Port 22
```

Expected: `TcpTestSucceeded : True`

![POWERSHELL](images/step6.1.3.png)

**AWS → HQ:** SSH to the test VM (`ssh -p 2222 ubuntu@<FGT-EIP>`) and run:

```bash
nc -zv 192.168.2.10 3389
```

Expected: `Connection to 192.168.2.10 3389 port [tcp/ms-wbt-server] succeeded!`

<details>
<summary><b>Why no AWS route change was needed</b></summary>

The private route table still sends `0.0.0.0/0` to `port2`. Traffic to on-prem and to the Internet both reach FortiGate the same way. FortiGate's own routing table then sends `192.168.0.0/22` into the tunnel, because that route is more specific. You don't need to change AWS route tables, add a Transit Gateway, or use an AWS VPN Gateway. Traffic across the tunnel isn't NATed, so each side sees the other's real IPs, and both FortiGates log every flow.
</details>

---

## Troubleshooting

| Issue | Solution |
| --- | --- |
| Tunnel stays down | Check the PSK (case-sensitive), confirm the SG sources match the on-prem `/32`, and confirm NAT-T is enabled |
| `no proposal chosen` | Both sides must use the same IKE version and wizard defaults |
| Phase 2 fails (`INVALID_ID_INFORMATION`) | Local and remote subnets must mirror each other |
| Tunnel up, no traffic | Check that both wizard policies (`port2 ↔ tunnel`) are enabled and that the static route exists |
| Tunnel drops when idle | Confirm NAT-T keepalive is `10` s on both sides |

Packet capture on the AWS FortiGate:

```text
diagnose sniffer packet any "host 10.100.2.10 and host 192.168.2.10" 4 20
```

---

## Clean-Up

Delete the resources in this order. Some resources depend on others.

1. **EC2 → Instances:** terminate `redwood-aws101-lab-fgt` and `redwood-aws101-lab-testvm`
2. **EC2 → Elastic IPs:** release `redwood-aws101-lab-fgt-eip`
3. **EC2 → Network Interfaces:** delete `redwood-aws101-lab-fgt-eni-port2` if it still exists
4. **EC2 → Security Groups:** delete `redwood-aws101-lab-testvm-sg` and `redwood-aws101-lab-fgt-sg`
5. **EC2 → Key Pairs:** delete `redwood-aws101-lab-kp` and the local key file
6. **VPC → Your VPCs:** delete `redwood-aws101-lab-vpc` (this also removes its subnets, route tables, and IGW)
7. **Resource Groups & Tag Editor:** delete `redwood-aws101-lab-rg` (if you created it in Lab 1). Then use **Tag Editor** to search all Regions for `Project=Redwood-AWS-101` and confirm nothing remains.
8. *(Optional)* On your HQ FortiGate, remove the `to_aws` tunnel and its policies and routes. The HQ environment is hosted by your instructor and is not in your AWS account.

An unattached Elastic IP, a stopped instance's EBS volumes, or a forgotten instance keeps billing. Clean-up is complete only when the Tag Editor search returns nothing.

---

## Workshop Complete

Redwood now has a hybrid environment with FortiGate enforcing one security policy everywhere:

| Lab | What you built |
| --- | --- |
| 1 | An AWS landing zone: VPC, public and private subnets, Internet Gateway |
| 2 | FortiGate as the only path in and out, using route tables and source/dest. check |
| 3 | A published application server, inspected and logged in both directions |
| 4 | An encrypted, inspected connection to HQ, with no AWS routing changes |

**Where to go next:**

- **AWS-102:** FortiGate HA across two Availability Zones (removes the single point of failure)
- **AWS-103:** Transit Gateway and GWLB designs for multi-VPC and east-west inspection

**Congratulations!**
