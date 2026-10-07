# AWS-101 Lab 2: FortiGate VM Deployment & Traffic Steering

## Overview

> **Scenario:** Redwood's security team requires FortiGate to be the only path in and out of AWS workloads, just as at HQ. You deploy it now, before any workload exists.

Deploy a FortiGate-VM with two interfaces (`port1` public, `port2` private), license it with FortiFlex, and route the private subnet through FortiGate.

**Prerequisites:** Lab 1 completed, plus the FortiFlex token and SSH client from [What You Need](/README.md#what-you-need).

![REFERENCE ARCHITECTURE](images/reference-architecture-lab2.png)

---

## PART 1: FortiGate VM Deployment

## Step 1: Subscribe to the FortiGate-VM AMI

1. Search for `Marketplace`, open **AWS Marketplace**, and confirm the region is **ca-central-1**.

   ![MARKETPLACE](images/step1.1.png)

2. Click **Discover products** and search for `Fortinet FortiGate Next-Generation Firewall`.

   ![DISCOVER](images/step1.2.png)

3. Select the **Fortinet, Inc.** listing **without "(PAYG)"** in the title. That is the BYOL listing. PAYG bills the license hourly through AWS. BYOL uses your own license (FortiFlex) and has no software charge. Use the **x86_64** version to match `c5.large`.

   ![RIGHT PRODUCT](images/step1.3.png)
<!-- TODO: verify current Marketplace listing titles and architectures against the current AWS Marketplace -->

4. Click **View purchase options → Subscribe** and accept the EULA.

   ![PURCHASE](images/step1.4.png)

5. Do **not** use the Marketplace launch wizard. You will launch the instance from EC2 in Step 3.

**Validate:** **Manage subscriptions** lists the FortiGate listing as active.

![VALIDATION](images/step1.validation.png)

---

## Step 2: Create the EC2 Key Pair

1. Open **EC2 → Key Pairs** and click **Create key pair**:

   ![KEY PAIRS](images/step2.1.png)

   | Parameter | Value |
   | --- | --- |
   | Name | `redwood-aws101-lab-kp` |
   | Key pair type | `RSA` |
   | Private key file format | `.pem` (macOS/Linux/WSL) or `.ppk` (PuTTY) |
   | Tag | `Project` = `Redwood-AWS-101` |

   ![CREATE KEY PAIR](images/step2.2.png)

2. The private key downloads only once, so keep it safe. On macOS/Linux/WSL, run:

   ```bash
   mkdir -p ~/.ssh/aws-101
   mv ~/Downloads/redwood-aws101-lab-kp.pem ~/.ssh/aws-101/
   chmod 400 ~/.ssh/aws-101/redwood-aws101-lab-kp.pem
   ```

   ![VERIFICATION](images/step2.4.png)

---

## Step 3: Launch the FortiGate EC2 Instance

1. Open **EC2 → Instances → Launch instances**.

   ![LAUNCH INSTANCE](images/step3.1.png)

2. **Name and tags:** click **Add additional tags**:

   | Parameter | Value |
   | --- | --- |
   | Name | `redwood-aws101-lab-fgt` |
   | Tag | `Project` = `Redwood-AWS-101` |
   | Resource types | **Instances, Volumes, Network interfaces** |

   ![TAGS](images/step3.2.png)

3. **AMI:** click **Browse more AMIs → AWS Marketplace AMIs**, search for `FortiGate BYOL`, and select the **Fortinet FortiGate (BYOL)** listing.

   ![SELECT AMI](images/step3.3.png)

4. **Instance type:** `c5.large`

   ![INSTANCE TYPE](images/step3.4.png)
<!-- TODO: verify against the current FortiGate-VM on AWS supported instance types list -->

5. **Key pair:** `redwood-aws101-lab-kp`

6. **Network settings:** click **Edit**:

   | Parameter | Value |
   | --- | --- |
   | VPC | `redwood-aws101-lab-vpc` |
   | Subnet | `redwood-aws101-lab-subnet-public-1a` |
   | Auto-assign public IP | **Disable** |
   | Firewall (security groups) | **Create security group** |
   | Security group name | `redwood-aws101-lab-fgt-sg` |
   | Description | `Management and inspection access for redwood-aws101-lab-fgt` |

   ![NETWORK SETTINGS](images/step3.6.a.png)

   Replace the default inbound rule with:

   | Type | Port range | Source | Description |
   | --- | --- | --- | --- |
   | SSH | 22 | My IP | FortiGate CLI |
   | HTTPS | 443 | My IP | FortiGate GUI |
   | Custom TCP | 2222 | `0.0.0.0/0` | Incoming access to server |
   | Custom TCP | 8080 | `0.0.0.0/0` | Incoming access to server |
   | All traffic | All | `10.100.0.0/16` | Traffic from VPC |

   ![SG GROUP CONFIG I](images/step3.6.b.gif)
   ![SG GROUP CONFIG II](images/step3.6.d.png)

> [!IMPORTANT]
> Keep **Auto-assign public IP** disabled. You will attach an Elastic IP in Step 4. An auto-assigned IP changes when the instance stops.

<details>
<summary><b>Security group rules explained</b></summary>

- **22 / 443 from My IP:** FortiGate CLI and GUI management, limited to your workstation
- **2222 / 8080 from anywhere:** inbound VIPs to the test VM (Lab 3)
- **All traffic from `10.100.0.0/16`:** traffic from workloads entering `port2` for inspection

In production, use a separate security group for each ENI role (Internet-facing vs. internal) and limit management to a trusted admin CIDR.
</details>

7. **Storage:** confirm the AMI provides a 2 GiB root volume and a 30 GiB log volume (`/dev/sdb`). Add the log volume only if it is missing.
   <!-- TODO: verify the default block-device mapping of the current FortiGate BYOL AMI -->

   Under **Advanced details**, set **Metadata version** to **V2 only (token required)**. IMDSv2 protects instance credentials from SSRF attacks.
   <!-- TODO: verify IMDSv2-only support for the FortiOS version in use -->

8. Click **Launch instance**. Wait until the instance is **Running** with **3/3 checks passed**.

   ![LAUNCH INSTANCE](images/step3.8.png)
   ![RUNNING](images/step3.9.png)

---

## Step 4: Allocate an Elastic IP for `port1`

An Elastic IP is a persistent public address that belongs to your account. FortiGate needs a stable address for management, the Lab 3 VIPs, and the Lab 4 VPN peer.

1. Open **EC2 → Elastic IPs → Allocate Elastic IP address**, use the parameters below, and click **Allocate**:

   | Parameter | Value |
   | --- | --- |
   | Public IPv4 address pool | Amazon's pool of IPv4 addresses |
   | Tag | `Name` = `redwood-aws101-lab-fgt-eip` |
   | Tag | `Project` = `Redwood-AWS-101` |

   ![ALLOCATE EIP](images/step4.1.a.png)
   ![ALLOCATE](images/step4.1.b.png)

2. Select the EIP and click **Actions → Associate Elastic IP address**:

   ![ASSOCIATE EIP](images/step4.2.a.png)

   | Parameter | Value |
   | --- | --- |
   | Resource type | **Network interface** |
   | Network interface | the primary ENI of `redwood-aws101-lab-fgt` |
   | Private IP address | the `10.100.1.x` address |

   ![ASSOCIATE](images/step4.2.b.png)

3. **Write down the Elastic IP.** You need it in every remaining lab, where it appears as `<FGT-EIP>`.

---

## Step 5: Create the `port2` Network Interface

1. Open **EC2 → Network Interfaces → Create network interface**:

   ![CREATE ENI](images/step5.1.png)

   | Parameter | Value |
   | --- | --- |
   | Description | `redwood-aws101-lab-fgt port2 (internal inspection interface)` |
   | Subnet | `redwood-aws101-lab-subnet-private-1a` |
   | Interface type | **ENA** |
   | Private IPv4 address | **Custom** → `10.100.2.4` |
   | Security groups | `redwood-aws101-lab-fgt-sg` |
   | Name tag | `redwood-aws101-lab-fgt-eni-port2` |
   | Tag | `Project` = `Redwood-AWS-101` |

   ![CREATE](images/step5.2.gif)

> [!IMPORTANT]
> `port2` must use the static IP `10.100.2.4`. The private route table points at this interface.

You create the ENI separately because an instance can have only one network interface at launch. Extra ENIs are standalone objects, so you can move one to a replacement instance.

---

## Step 6: Attach `port2` to FortiGate

1. Select `redwood-aws101-lab-fgt` and click **Instance state → Stop instance**. Wait until it shows **Stopped**. This makes sure FortiOS detects the new interface.

   ![STOP INSTANCE](images/step6.1.png)

2. Click **Actions → Networking → Attach network interface**, select the `port2` interface, and click **Attach**.

   ![ATTACH ENI](images/step6.2.a.png)
   ![ATTACH](images/step6.2.b.png)

3. On the **Networking** tab, confirm two interfaces: `port1` in the public subnet and `port2` at `10.100.2.4`.

   ![VERIFY](images/step6.3.gif)

---

## Step 7: Disable Source/Destination Check

AWS drops traffic that an instance forwards on behalf of other hosts unless this check is disabled.

<details>
<summary><b>What is the source/destination check?</b></summary>

By default, an ENI accepts only packets whose source or destination is its own IP. A firewall forwards traffic for other hosts: for example, the source is the test VM and the destination is the Internet. With the check enabled, AWS silently drops that traffic. Forgetting this is the most common reason a FortiGate looks healthy but passes no traffic.
</details>

1. In **Network Interfaces**, select FortiGate's primary ENI. Click **Actions → Change source/dest. check**, uncheck it, and click **Save**.

   ![UNCHECK SRC/DST CHECKING](images/step7.1.gif)

2. Repeat for `redwood-aws101-lab-fgt-eni-port2`.

3. Start `redwood-aws101-lab-fgt` and wait for **3/3 checks passed**.

   ![START INSTANCE](images/step7.3.png)

---

## Step 8: License FortiGate and Access the GUI

FortiGate starts with a limited evaluation license (1 vCPU, 2 GB RAM). Activating FortiFlex unlocks the full VM and FortiGuard services.

1. Browse to `https://<FGT-EIP>` and accept the self-signed certificate warning.

2. Log in as `admin`. The password is the **EC2 instance ID** (e.g. `i-0abc123def4567890`).

3. Set a new admin password when prompted.

4. In the **VM License** dialog, select **FortiFlex token**, paste your token, click **OK**, and confirm the reboot.

   ![FORTIFLEX](images/step8.4.png)

5. Log in again and complete the setup prompts. Under **System → FortiGuard**, confirm that the license is **Valid**.

   ![LICENSES](images/step8.5.png)

6. Under **Network → Interfaces**, confirm `port1` (`10.100.1.x`) and `port2` (`10.100.2.4`) are **Up**.

   ![INTERFACES](images/step8.6.a.png)

   If `port2` has no address, set it statically to `10.100.2.4/24`:

   ![CONFIG PORT2](images/step8.6.b.png)

> [!TIP]
> FortiGate shows the private `10.100.1.x` address on `port1`. The Elastic IP is applied by AWS and is never visible inside FortiOS.

---

## PART 2: Traffic Steering

## Step 9: Create the Private Route Table

This route table makes FortiGate the inspection point: any traffic leaving the private subnet goes to `port2`. Without it, the private subnet uses the Main route table (only the `local` route) and can't leave the VPC.

1. Open **VPC → Route tables → Create route table**:

   ![ROUTE TABLES](images/step9.1.png)

   | Parameter | Value |
   | --- | --- |
   | Name | `redwood-aws101-lab-rt-private` |
   | VPC | `redwood-aws101-lab-vpc` |
   | Tag | `Project` = `Redwood-AWS-101` |

   ![CREATE ROUTE TABLE](images/step9.1.b.png)

2. Click **Routes → Edit routes → Add route**, then **Save changes**:

   | Destination | Target |
   | --- | --- |
   | `0.0.0.0/0` | **Network Interface** → `redwood-aws101-lab-fgt-eni-port2` |

   ![ADD ROUTE](images/step9.2.png)

> [!IMPORTANT]
> Choose the **Network Interface** target, not **Instance**.

<details>
<summary><b>Why target the ENI?</b></summary>

VPC routes point to AWS objects (ENIs, gateways, endpoints), never to a bare IP address. FortiGate has two ENIs, so targeting the `port2` ENI makes the next hop explicit. If the instance is replaced, the route still works once the same ENI is reattached.
</details>

3. Click **Subnet associations → Edit subnet associations**, select `redwood-aws101-lab-subnet-private-1a`, and click **Save associations**.

   ![SUBNET ASSOCIATION](images/step9.3.gif)

<details>
<summary><b>Production considerations</b></summary>

- **Availability:** a single FortiGate in one AZ is a single point of failure. In production, use FortiGate FGCP active-passive HA across two AZs (AWS-102) or a FortiGate fleet behind a Gateway Load Balancer.
- **Sizing:** choose the instance type based on the throughput you need with security profiles enabled. Check Fortinet's list of supported instance types.
</details>

---

## Checklist

- [ ] `redwood-aws101-lab-fgt` is **Running**, with `port1` (with the Elastic IP) and `port2` (`10.100.2.4`)
- [ ] Source/dest. check is **disabled** on both interfaces
- [ ] FortiGate GUI is reachable and the FortiFlex license is **Valid**
- [ ] `redwood-aws101-lab-rt-private` has `0.0.0.0/0 → port2 ENI` and is associated with the private subnet

## Troubleshooting

| Issue | Solution |
| --- | --- |
| GUI times out | The SG needs HTTPS from your **current** IP (it may have changed). |
| Login fails | The initial password is the **EC2 instance ID**. |
| `port2` IP is not `10.100.2.4` | Delete the ENI and recreate it with a **Custom** IP (Step 5). |
| FortiFlex: "Failed to contact FortiCare" | Check the public route table (Lab 1, Step 6). Test with `execute ping fortiguard.com`. |
| FortiFlex: "Token already used" | Ask your instructor for a new token. |

---

FortiGate now controls every path in and out of the private subnet, and it blocks everything until you add policies. In Lab 3, Redwood's first application server arrives.

*Next:* [**Lab 3: Security Policies & Traffic Testing**](/aws-101-lab3/README.md)
