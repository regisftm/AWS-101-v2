# AWS-101 Lab 1: AWS Infrastructure Foundation

## Overview

> **Scenario:** Before Redwood can deploy anything in AWS, it needs a network for FortiGate to protect. Your first job is to build the landing zone.

Build a VPC, a public and a private subnet, an Internet Gateway, and the public subnet's route table. The private subnet's route table is created in Lab 2, once FortiGate exists.

**Prerequisites:** the AWS account described in [What You Need](/README.md#what-you-need). Sign in as an MFA-protected IAM identity, never the root user.

![REFERENCE ARCHITECTURE](images/ref-architecture-lab1.png)

---

## Naming & Tagging

Resources are named `redwood-aws101-lab-<resource>` (zonal resources end with the AZ, e.g. `-1a`).

Apply these **standard tags** to every resource. The console sets `Name` automatically from the **Name** field.

| Tag Key | Tag Value |
| --- | --- |
| `Project` | `Redwood-AWS-101` |
| `Environment` | `lab` |
| `Owner` | `<your-name>` |

The `Project` tag drives the Resource Group, cost reports, and clean-up. `Owner` identifies who created a resource in a shared account.

---

## Step 1: Create an AWS Resource Group (Optional)

A tag-based Resource Group lists every workshop resource in one place, which makes clean-up easier.

> [!NOTE]
> Do not block pop-ups for the AWS Management Console.

1. Sign in to <https://console.aws.amazon.com> and select **Canada (Central) ca-central-1** in the top-right corner. All work happens in this region.

   ![REGION](images/step1.1.png)

2. Search for `Resource Groups` and open **Resource Groups & Tag Editor**.

   ![RESOURCE GROUPS](images/step1.2.png)

3. Click **Create resource group**:

   ![CREATE RESOURCE GROUP](images/step1.3.a.png)

   | Parameter | Value |
   | --- | --- |
   | Group type | Tag based |
   | Resource types | All supported resource types |
   | Tag Key / Value | `Project` / `Redwood-AWS-101` |
   | Group name | `redwood-aws101-lab-rg` |
   | Group tags | standard tags |

   ![RG PARAMETERS I](images/step1.3.b.png)
   ![RG PARAMETERS II](images/step1.3.c.png)

4. Click **Create group**.

   ![RG CREATED](images/step1.4.png)

---

## Step 2: Create the VPC

1. Search for `VPC`, open the **VPC** console, and confirm the region is **ca-central-1**.

   ![VPC](images/step2.1.png)

2. Click **Your VPCs → Create VPC**.

   ![CREATE VPC](images/step2.2.png)

3. Use the parameters below and add the standard tags:

   | Parameter | Value |
   | --- | --- |
   | Resources to create | **VPC only** |
   | Name tag | `redwood-aws101-lab-vpc` |
   | IPv4 CIDR | `10.100.0.0/16` |
   | IPv6 CIDR block | No IPv6 CIDR block |
   | Tenancy | Default |

   ![VPC PARAMS](images/step2.3.png)
   ![VPC TAG + CREATE](images/step2.4.png)

4. Click **Create VPC**.

   ![VPC INFORMATION](images/step2.6.png)

<details>
<summary><b>Why <code>10.100.0.0/16</code>?</b></summary>

- RFC 1918 private range with 65,536 addresses
- Does not overlap the on-prem network (`192.168.0.0/22`), which the VPN in Lab 4 requires
- Leaves room for more subnets or VPCs (`10.101.0.0/16`, ...)
- We skip **VPC and more** so that each subnet and route table is built explicitly
</details>

---

## Step 3: Create the Public Subnet

This subnet hosts FortiGate's `port1` (Internet-facing).

1. Click **Subnets → Create subnet**.

   ![CREATE SUBNET](images/step3.1.png)

2. Use the parameters below and add the standard tags:

   | Parameter | Value |
   | --- | --- |
   | VPC ID | `redwood-aws101-lab-vpc` |
   | Subnet name | `redwood-aws101-lab-subnet-public-1a` |
   | Availability Zone | `ca-central-1a` |
   | IPv4 subnet CIDR block | `10.100.1.0/24` |

   ![SUBNET PARAMS I](images/step3.2.png)
   ![SUBNET PARAMS II](images/step3.3.png)

3. Click **Create subnet**.

> [!IMPORTANT]
> Both subnets must be in `ca-central-1a`. An ENI can only attach to an instance in its own AZ.

<details>
<summary><b>AWS reserved IPs: why 251 usable addresses?</b></summary>

AWS reserves 5 addresses in every subnet:

| Address | Use |
| --- | --- |
| `10.100.1.0` | Network address |
| `10.100.1.1` | VPC router (default gateway) |
| `10.100.1.2` | Reserved (the VPC DNS resolver is at `10.100.0.2`) |
| `10.100.1.3` | Reserved for future use |
| `10.100.1.255` | Broadcast (not supported in a VPC) |

Usable addresses: `10.100.1.4` to `10.100.1.254`.
</details>

---

## Step 4: Create the Private Subnet

This subnet hosts FortiGate's `port2` and the protected workloads.

1. Click **Create subnet** again, use the parameters below, and add the standard tags:

   | Parameter | Value |
   | --- | --- |
   | VPC ID | `redwood-aws101-lab-vpc` |
   | Subnet name | `redwood-aws101-lab-subnet-private-1a` |
   | Availability Zone | `ca-central-1a` |
   | IPv4 subnet CIDR block | `10.100.2.0/24` |

2. Click **Create subnet**.

   ![SUBNETS](images/step4.3.png)

In Lab 2, this subnet's route table will send `0.0.0.0/0` to FortiGate `port2` (`10.100.2.4`), so all workload traffic is inspected.

```text
Outbound: Workload → port2 (inspect, NAT) → port1 → IGW → Internet
Inbound:  Internet → IGW → port1 (inspect, NAT) → port2 → Workload
```

<details>
<summary><b>Why only two subnets?</b></summary>

This is Fortinet's reference design for a single FortiGate-VM on AWS: a **Public** subnet for `port1` and a **Private** subnet shared by `port2` and the workloads. A separate transit subnet would not add security here, because subnet route tables only affect traffic that leaves the subnet. Dedicated transit, HA-sync, and management subnets matter in HA, GWLB, and Transit Gateway designs (AWS-102/103).
</details>

<details>
<summary><b>East-west traffic in the same subnet is NOT inspected</b></summary>

Two instances in the same subnet talk directly through the VPC's implicit `local` route. AWS doesn't let any route table override that route within a subnet. This is an AWS limitation, not a FortiGate one. To inspect east-west traffic, use separate workload subnets with FortiGate as the next hop, a Gateway Load Balancer, or a Transit Gateway inspection VPC (covered in AWS-103).
</details>

---

## Step 5: Create and Attach an Internet Gateway

A VPC has no Internet access until an Internet Gateway (IGW) is attached **and** a route table points to it. FortiGate's Elastic IP (Lab 2) depends on the IGW.

1. Click **Internet gateways → Create internet gateway**.

   ![IGW](images/step5.1.png)

2. Set **Name tag** to `redwood-aws101-lab-igw`, add the standard tags, and click **Create internet gateway**.

   ![CREATE IGW](images/step5.2.png)

3. Click **Actions → Attach to VPC**, select `redwood-aws101-lab-vpc`, and click **Attach internet gateway**.

   ![ATTACH IGW](images/step5.3.png)
   ![ATTACH IGW TO VPC](images/step5.3.b.png)

---

## Step 6: Create the Public Route Table

A subnet becomes "public" only when its route table has `0.0.0.0/0 → IGW`. FortiGate's `port1` needs this route for licensing, FortiGuard updates, and egress.

> [!NOTE]
> Give each subnet its own route table and leave the VPC's **Main** route table with only the `local` route. A subnet you forget to associate then can't route anywhere, rather than inheriting a permissive route.

1. Click **Route tables → Create route table**:

   ![OPEN RT](images/step6.1.png)

   | Parameter | Value |
   | --- | --- |
   | Name | `redwood-aws101-lab-rt-public` |
   | VPC | `redwood-aws101-lab-vpc` |
   | Tags | standard tags |

   ![CREATE RT](images/step6.1.b.png)

2. Open the route table, go to **Routes → Edit routes → Add route**, and click **Save changes**:

   ![ROUTES](images/step6.2.a.png)

   | Destination | Target |
   | --- | --- |
   | `0.0.0.0/0` | Internet Gateway → `redwood-aws101-lab-igw` |

   ![ADD ROUTE](images/step6.2.b.png)

3. Go to **Subnet associations → Edit subnet associations**, select `redwood-aws101-lab-subnet-public-1a`, and click **Save associations**.

   ![EDIT SUBNET ASSOCIATIONS](images/step6.3.a.png)
   ![SAVE ASSOCIATIONS](images/step6.3.b.png)

---

## Checklist

- [ ] VPC `redwood-aws101-lab-vpc` (`10.100.0.0/16`) in `ca-central-1`
- [ ] Public subnet `10.100.1.0/24` and private subnet `10.100.2.0/24`, both in `ca-central-1a`
- [ ] IGW `redwood-aws101-lab-igw` is **Attached** to the VPC
- [ ] `redwood-aws101-lab-rt-public` has `0.0.0.0/0 → redwood-aws101-lab-igw` and is associated with the public subnet

## Troubleshooting

| Issue | Solution |
| --- | --- |
| Overlapping CIDR error | Another VPC or subnet already uses that range. Check the CIDR values. |
| Resources in the wrong region | Delete them and recreate in `ca-central-1` |
| IGW will not attach | An IGW can attach to only one VPC. Detach it or create a new one. |

---

The landing zone is ready, but the private subnet has no way out yet. In Lab 2, FortiGate becomes that way out.

*Next:* [**Lab 2: FortiGate VM Deployment & Traffic Steering**](/aws-101-lab2/README.md)
