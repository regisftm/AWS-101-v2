# AWS-101 Lab 3: Security Policies & Traffic Testing

## Lab Overview

### Objective

Deploy a test workload (EC2 instance) into `Private-Subnet`, then configure FortiGate firewall policies and Virtual IPs (VIPs) so that:

- Inbound SSH and HTTP traffic from the Internet reaches the workload through FortiGate
- Outbound traffic from the workload to the Internet is inspected and source-NAT'd to FortiGate's Elastic IP

By the end of this lab, you'll have proven end-to-end inspection in both directions, watched the sessions appear in **Log & Report → Forward Traffic** and **FortiView**, and demonstrated that nothing reaches or leaves `Private-Subnet` without FortiGate's permission.

### What You'll Build

- A test EC2 instance (`Redwood-AWS-TestVM`) running Ubuntu Server 26.04 LTS in `Private-Subnet` at the static private IP `10.100.2.10`, with **no public IP**
- A FortiGate **address object** (`TESTVM-INTERNAL`) representing the workload
- Two FortiGate **Virtual IPs** mapping FortiGate's Elastic IP to the test VM:
  - `TESTVM-INTERNAL-VIP-SSH` — external port `2222` → internal port `22`
  - `TESTVM-INTERNAL-VIP-HTTP` — external port `8080` → internal port `80`
- A **Virtual IP Group** (`TESTVM-INTERNAL-VIPGRP`) aggregating both VIPs
- A FortiGate firewall policy (`testvm_access_vip`) — `port1 → port2`, allowing inbound SSH/HTTP via the VIPs
- A FortiGate firewall policy (`internet_access`) — `port2 → port1`, allowing outbound Internet access with **source NAT** to FortiGate's Elastic IP
- Validation that traffic appears in **Forward Traffic logs** and **FortiView Sources** in real time

### Architecture After Lab 3

![REFERENCE ARCHITECTURE](images/reference-architecture-lab3.png)

### Business Context

Redwood Industries has approved the AWS landing zone built in Labs 1 and 2 and is ready to deploy its first workload — a test application server. Before any production workload lands in AWS, the security team requires proof that all traffic is inspected by FortiGate, that workloads cannot reach the Internet directly, and that inbound access is controlled at the firewall (not by a VPC-wide allow-all). In this lab you will deploy the test workload, configure the inbound and outbound policies that satisfy that posture, and demonstrate the inspection by generating real traffic and watching it land in FortiGate's logs and FortiView.

---

## Prerequisites

- Lab 1 and Lab 2 completed (VPC, subnets, IGW, Public RT, FortiGate VM with port1+port2 ENIs, FortiFlex licence valid, `Redwood-AWS-RT-Private` associated with `Private-Subnet`)
- The Elastic IP from Lab 2 — you will use it as the SSH endpoint for the test workload
- Your existing `Redwood-AWS-FGT-Key` key pair (created in Lab 2) — the test VM will use the same key
- An SSH client (macOS / Linux / WSL terminal, or PuTTY on Windows)

> [!IMPORTANT]
> The test VM is deliberately deployed **without** a public IP. The only inbound path is the FortiGate VIP. If you forget to disable "Auto-assign public IPv4" during launch, AWS will hand the VM a temporary public IP and traffic will bypass FortiGate — which defeats the entire purpose of this lab.

---

## PART 1: Deploy the Test Workload

## Step 1: Launch the Test EC2 Instance

You will launch a small Ubuntu Server 26.04 LTS instance directly into `Private-Subnet`. The `Redwood-AWS-RT-Private` route table from Lab 2 ensures any egress from this instance is automatically steered to FortiGate's `port2`.

1. **Open the EC2 launch wizard:**
   - In the top search bar, type `EC2` and click **EC2** (the service result)
   - In the left navigation menu, click **Instances**
   - Click **Launch instances**

2. **Name and tags:**
   - Use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `Redwood-AWS-TestVM` |
     | Tags | |
     | `Project` | `Redwood-AWS-101` |

     ![TAGS](images/step1.2.png)

3. **Application and OS Images (Amazon Machine Image):**
   - Under **Quick Start**, select **Ubuntu**
   - In the **Amazon Machine Image (AMI)** dropdown, select **Ubuntu Server 26.04 LTS (HVM), SSD Volume Type**
   - Architecture: `64-bit (x86)`

     ![OS IMAGE](images/step1.3.png)

4. **Instance type:**
   - Use the parameter below.

     | Parameter | Value |
     | --- | --- |
     | Instance type | `t2.micro` (1 vCPU, 1 GiB, free-tier eligible) |

5. **Key pair (login):**
   - In the **Key pair name — required** dropdown, select `Redwood-AWS-FGT-Key` (created in Lab 2)
   - You can reuse the same key — the test VM is for short-lived workshop traffic generation, not a production deployment

6. **Network settings:**
   - Click **Edit** to expand the panel and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | VPC | `Redwood-AWS-VPC` |
     | Subnet | `Private-Subnet` |
     | Auto-assign public IP | **Disable** |
     | Firewall (security groups) | **Create security group** |
     | Security group name | `Redwood-AWS-TestVM-SG` |
     | Description | `Private access only - inbound from FortiGate port2 ENI` |

     ![NETWORK](images/step1.6.a.png)

   - In the **Inbound security group rules** section, replace the default rule with the two rules below.

     | Type | Protocol | Port range | Source type | Source | Description |
     | --- | --- | --- | --- | --- | --- |
     | SSH | TCP | 22 | Anywhere | 0.0.0.0/0 | SSH from Internet to VM (via FortiGate) |
     | HTTP | TCP | 80 | Anywhere | 0.0.0.0/0 | HTTP from Internet to VM (via FortiGate) |

     ![SG](images/step1.6.b.png)

7. **Advanced network configuration:**
   - Expand the **Advanced network configuration** panel
   - In **Network interface 1** find the **Primary IP** field.
   - Set the **Primary IP** private IPv4 to `10.100.2.10`

     ![FIXED IP ADD](images/step1.7.png)

8. **Configure storage:**
   - Keep the default 8 GiB `gp3` root volume (no second volume needed)

9. **Review and launch:**
   - Go to the **Summary** panel on the right
   - Confirm **Number of instances** is `1`
   - Click **Launch instance**

10. **Wait for the instance to reach `Running` state:**
    - Click **View all instances**
    - Refresh the **Instances** page until the **Instance state** shows **Running** and the **Status check** column shows **2/2 checks passed**

     ![RUNNING INSTANCE](images/step1.10.png)

### Validation

- [x] `Redwood-AWS-TestVM` appears in the **Instances** list with state **Running**
- [x] **Networking** tab shows the primary ENI is in `Private-Subnet` with private IP `10.100.2.10` and **no public IPv4 address**
- [x] Security group `Redwood-AWS-TestVM-SG` is attached and contains SSH (22) and HTTP (80) rules sourced from `0.0.0.0/0`
- [x] Tag `Project = Redwood-AWS-101` is present

---

## PART 2: Configure FortiGate for Inbound Access (VIPs)

The test VM is now running but unreachable from the outside — it has no public IP, and even FortiGate doesn't yet have policies that recognize it. This part configures FortiGate so that traffic from the Internet to FortiGate's Elastic IP on specific ports gets translated and forwarded to `Redwood-AWS-TestVM`.

## Step 2: Create the FortiGate Address Object for the Test Workload

Address objects are reusable references that FortiGate uses in policies, VIPs, and routes. Defining the test VM as a named object once makes every subsequent policy easier to read.

1. **Log into FortiGate:**
   - In your browser, navigate to `https://<FortiGate-Elastic-IP>`
   - Log in as `admin` with the password you set in Lab 2

2. **Create the address object:**
   - In the left navigation menu, click **Policy & Objects → Addresses**
   - Click **+ Create new** in the **Address** tab and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `TESTVM-INTERNAL` |
     | Interface | `port2` |
     | Type | `Subnet` |
     | IP/Netmask | `10.100.2.10/32` |
     | Routing configuration | `Disabled` |

     ![ADDRESS CONFIG](images/step2.2.png)

   - Click **OK**

### Validation

- [x] `TESTVM-INTERNAL` appears in **Policy & Objects → Addresses** with type `Subnet` and IP value `10.100.2.10/32`
- [x] Interface shows `port2`

---

## Step 3: Create the Virtual IPs for SSH and HTTP

Virtual IPs (VIPs) are FortiGate's destination NAT mechanism. Each VIP maps a `(public IP, public port)` tuple to a `(private IP, private port)` tuple. You will create two VIPs — one for SSH, one for HTTP — both targeting `Redwood-AWS-TestVM`.

> [!NOTE]
> The **Public IP address** is set to `0.0.0.0` rather than the FortiGate Elastic IP. This is the standard pattern for AWS-deployed FortiGates — `port1` carries a private VPC address (e.g., `10.100.1.x`), while the Elastic IP is provided by the IGW via 1:1 NAT (transparent to FortiOS). Setting the External IP to `0.0.0.0` instructs FortiGate to use the actual `port1` interface IP at runtime, which AWS then 1:1-NATs from the Elastic IP.

1. **Create the SSH VIP:**
   - In the left navigation menu, click **Policy & Objects → Virtual IPs**
   - Click **+ Create new → Virtual IP** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `TESTVM-INTERNAL-VIP-SSH` |
     | **Network** | |
     | Interface | `port1` |
     | Type | `Static NAT` |
     | External IP address/range | `0.0.0.0` |
     | Map to IPv4 address/range | `TESTVM-INTERNAL` (select from suggestions) |
     | **Port Forwarding** | |
     | Port Forwarding | **Enabled** |
     | Protocol | `TCP` |
     | Port Mapping Type | `One to one` |
     | External service port | `2222` |
     | Map to IPv4 port | `22` |

     ![VIP-SSH](images/step3.1.png)

   - Click **OK**

2. **Create the HTTP VIP:**
   - Still on **Policy & Objects → Virtual IPs**, click **+ Create new** in the **Virtual IP** tab and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `TESTVM-INTERNAL-VIP-HTTP` |
     | **Network** | |
     | Interface | `port1` |
     | Type | `Static NAT` |
     | External IP address/range | `0.0.0.0` |
     | Map to IPv4 address/range | `TESTVM-INTERNAL` (select from suggestions) |
     | **Port Forwarding** | |
     | Port Forwarding | **Enabled** |
     | Protocol | `TCP` |
     | Port Mapping Type | `One to one` |
     | External service port | `8080` |
     | Map to IPv4 port | `80` |

   - Click **OK**

### Validation

- [x] `TESTVM-INTERNAL-VIP-SSH` appears in **Virtual IPs**, mapping `0.0.0.0:2222 → 10.100.2.10:22`
- [x] `TESTVM-INTERNAL-VIP-HTTP` appears in **Virtual IPs**, mapping `0.0.0.0:8080 → 10.100.2.10:80`
- [x] Both VIPs have **Interface: port1**

![VALIDATION](images/step3.valid.png)

---

## Step 4: Create the Virtual IP Group

Grouping the two VIPs into a single object lets you reference both in one firewall policy instead of writing two parallel policies.

1. **Open the Virtual IP Groups tab:**
   - Still in **Policy & Objects → Virtual IPs**, switch to the **Virtual IP Group** tab at the top of the page
   - Click **+ Create new** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `TESTVM-INTERNAL-VIPGRP` |
     | Interface | `port1` |
     | Members | `TESTVM-INTERNAL-VIP-SSH`, `TESTVM-INTERNAL-VIP-HTTP` |

   - Click **OK**

     ![VIP GROUP](images/step4.1.png)

### Validation

- [x] `TESTVM-INTERNAL-VIPGRP` appears in the **Virtual IP Group** tab
- [x] Both VIPs are listed as members

---

## Step 5: Create the Inbound Firewall Policy (port1 → port2)

The VIPs do the address translation, but FortiGate still needs an explicit policy that allows the translated traffic to flow from `port1` to `port2`. By default, FortiGate denies all traffic — every allowed flow must match an explicit policy.

1. **Create the policy:**
   - In the left navigation menu, click **Policy & Objects → Firewall Policy**
   - Click **+ Create new** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `testvm_access_vip` |
     | Schedule | `always` |
     | Action | `ACCEPT` |
     | Incoming interface | `port1` |
     | Outgoing interface | `port2` |
     | **Source & Destination** | |
     | Source | `all` |
     | Destination | `TESTVM-INTERNAL-VIPGRP` |
     | Service | `SSH`, `HTTP` |
     | **Firewall/Network Options** | |
     | NAT | `Disabled` |
     | **Logging Options** | |
     | Log allowed traffic | `All sessions` |

   - Click **OK**

     ![FIREWALL POLICY CREATE](images/step5.1.png)

> [!NOTE]
> **Source = `all`** means any Internet host can attempt SSH or HTTP to the VIPs. For a production deployment, narrow this to the operator's office IP or a known jump-host CIDR. For the workshop the wide-open Source is intentional so attendees can connect from any network.

### Validation

- [x] `testvm_access_vip` appears in **Firewall Policy** with **Status: Enabled**
- [x] Policy direction is `port1 → port2`
- [x] Destination is `TESTVM-INTERNAL-VIPGRP`
- [x] Logging is set to **All sessions**

---

## Step 6: Test SSH Connectivity Through the VIP

You will now connect to the test VM from your workstation by SSH'ing to FortiGate's Elastic IP on port `2222`. FortiGate will receive the connection on `port1`, the SSH VIP will translate the destination to `10.100.2.10:22`, the firewall policy will permit it, and the packet will egress on `port2` to the test VM.

1. **SSH into the test VM via the FortiGate VIP:**
   - On macOS / Linux / WSL, run:

     ```bash
     ssh -i ~/.ssh/aws-101/Redwood-AWS-FGT-Key.pem -p 2222 ubuntu@<FortiGate-Elastic-IP>
     ```

   - On Windows / PuTTY, set:
     - Host name: `<FortiGate-Elastic-IP>`
     - Port: `2222`
     - Connection → SSH → Auth → Private key file: `Redwood-AWS-FGT-Key.ppk`
     - Login as: `ubuntu`

2. **Accept the host key warning the first time:**
   - When you see "The authenticity of host '... can't be established'", type `yes` and press Enter
   - You should land at the prompt: `[ubuntu@ip-10-100-2-10 ~]$`

     ![SSH ACCESS](images/step6.2.png)

> [!TIP]
> If the SSH connection hangs or refuses, check (in order): your local SSH client is reaching FortiGate on TCP/2222 (security group `Redwood-AWS-FGT-SG` must allow SSH from your IP — added in Lab 2 Step 3); the VIP and firewall policy you just created exist; the test VM is `Running`; and `Redwood-AWS-TestVM-SG` allows SSH from `0.0.0.0/16`.

3. **Confirm the test VM is reachable from inside the VPC but isolated from the Internet (yet):**
   - From the SSH session into the test VM, run:

     ```bash
     ping -c 3 10.100.2.4
     ```

   - You should get successful replies — this is FortiGate's `port2` ENI

   - Now try the Internet:

     ```bash
     ping -c 3 8.8.8.8
     ```

   - This will **fail** — packets leave the test VM, hit FortiGate's `port2`, but no firewall policy yet permits `port2 → port1`, so FortiGate drops them silently. This is the correct behaviour at this stage of the lab.

### Validation

- [x] SSH from your workstation to `<FortiGate-Elastic-IP>:2222` succeeds and lands at the test VM's prompt
- [x] `ping 10.100.2.4` from the test VM works (intra-VPC reachability)
- [x] `ping 8.8.8.8` from the test VM **fails** (no outbound policy yet — this is expected)

---

## PART 3: Configure FortiGate for Outbound Access

The test VM has inbound connectivity for management, but it can't reach the Internet for software updates, DNS, or any application traffic. You will now create the outbound firewall policy with source NAT so that egress traffic appears on the Internet as coming from FortiGate's Elastic IP.

---

## Step 7: Create the Outbound Internet Policy (port2 → port1) with NAT

1. **Create the policy:**
   - In FortiGate, navigate to **Policy & Objects → Firewall Policy**
   - Click **+ Create new** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `internet_access` |
     | Schedule | `always` |
     | Action | `ACCEPT` |
     | Incoming interface | `port2` |
     | Outgoing interface | `port1` |
     | **Source & Destination** | |
     | Source | `all` |
     | Destination | `all` |
     | Service | `ALL` |
     | **Firewall/Network Options** | |
     | NAT | `Enabled` |
     | IP Pool Configuration | `Use Outgoing Interface Address` |
     | **Logging Options** | |
     | Log allowed traffic | `All sessions` |

   - Click **OK**

     ![INTERNET ACCESS](images/step7.1.png)

> [!IMPORTANT]
> **NAT must be Enabled.** Without source NAT, the packet leaves FortiGate `port1` with source IP `10.100.2.10` (the test VM's private IP). The AWS Internet Gateway performs a source IP validation check — it will only forward packets whose source IP matches a public IP (Elastic IP) associated with the sending ENI. Since `10.100.2.10` has no associated EIP, **the IGW drops the packet silently**. With NAT enabled and **Use Outgoing Interface Address** selected, FortiGate replaces the source IP with `10.100.1.x` (`port1`'s private IP), which *does* have the Elastic IP associated with it. The IGW then performs its 1:1 NAT, substituting the Elastic IP as the public source, and forwards the packet to the Internet. Replies return to the Elastic IP → IGW → `port1` → FortiGate de-NAT → test VM.

### Validation

- [x] `internet_access` appears in **Firewall Policy** with **Status: Enabled**
- [x] Direction is `port2 → port1`
- [x] **NAT** column shows **Enabled** with **Use Outgoing Interface Address**
- [x] Logging is set to **All sessions**

---

## Step 8: Test Outbound Internet Access from the Test VM

1. **From the SSH session into the test VM, retest connectivity:**
   - ICMP:

     ```bash
     ping -c 3 8.8.8.8
     ```

     Expected: 0% packet loss, replies in ~10–30 ms.

   - DNS:

     ```bash
     ping -c 3 www.google.com
     ```

     Expected: name resolves to an IP, replies arrive.

   - HTTPS:

     ```bash
     curl -I https://www.fortinet.com
     ```

     Expected: `HTTP/2 200` response with headers.

   - A second host for confidence:

     ```bash
     curl -I https://www.amazon.ca
     ```

     Expected: `HTTP/2 200` response.

### Validation

- [x] `ping 8.8.8.8` succeeds (0% packet loss)
- [x] DNS resolves and `ping www.google.com` succeeds
- [x] `curl -I https://www.fortinet.com` returns `HTTP/2 200`
- [x] All Internet-bound traffic flows through FortiGate (will be confirmed in the next step)

---

## PART 4: Verify Inspection in FortiGate

Generating traffic is one half of the validation; confirming it actually traversed FortiGate is the other. Two FortiGate views give you that proof.

---

## Step 9: Inspect Forward Traffic Logs

1. **Open the Forward Traffic log:**
   - In the FortiGate GUI, click **Log & Report → Forward Traffic**
   - You should see fresh log entries from the connectivity tests in Step 8

   ![TRAFFIC LOG](images/step9.1.png)

2. **Examine a log entry — click any row to open the side details pane:**

   | Field | Expected value |
   | --- | --- |
   | Source IP | `10.100.2.10` (test VM) |
   | Source NAT IP | `10.100.1.x` (FortiGate `port1`) |
   | Destination | `8.8.8.8` (or whatever you tested) |
   | Source Interface | `port2` |
   | Destination Interface | `port1` |
   | Policy ID | `internet_access` |
   | Action | `accept` |
   | NAT Translation | `snat` |

   ![LOG DETAILS](images/step9.2.png)

> [!NOTE]
> In AWS, the Elastic IP does NOT appear in the FortiGate logs as the NAT'd source — you will see the `port1` interface address (`10.100.1.x`). The Elastic IP substitution happens at the IGW, downstream of FortiGate. To confirm the actual public-source-IP from outside the VPC, run `curl -s https://ifconfig.me` from the test VM and verify it returns the FortiGate Elastic IP configured in Lab 2.

### Validation

- [x] Forward Traffic log entries appear filtered by Source IP `10.100.2.10`
- [x] Policy column shows `internet_access` for outbound and `testvm_access_vip` for inbound flows
- [x] Action shows `accept` for all expected sessions
- [x] NAT translation visible on outbound entries (`port2 → port1`)

---

## Step 10: Visualize Traffic in FortiView

FortiView is FortiGate's real-time traffic dashboard. It needs no extra configuration — it draws from the same log stream you just inspected.

1. **Open FortiView Sources:**
   - In the FortiGate GUI, click **Dashboard → FortiView** and go to  **Sources** tap on the top.

     ![FORTIVIEW SOURCES](images/step10.1.png)

2. **Drill down on the test VM:**
   - Click the row for source `10.100.2.10`
   - Click **Drill down** → **Destination**

3. **Confirm the destinations and applications you tested:**
   - You should see `8.8.8.8` (ICMP), `www.fortinet.com` / `www.amazon.ca` (HTTPS), Google DNS (UDP/53)
   - Each session row shows bytes sent / bytes received, the matching policy, and the source/destination interfaces

     ![DRILL DOWN](images/step10.3.png)

### Validation

- [x] FortiView Sources lists `10.100.2.10` with non-zero session count and bytes
- [x] Destinations include the IPs and FQDNs you tested in Step 8
- [x] Sessions show realistic byte counts (HTTPS will show more bytes than ICMP)

---

## Lab 3 Complete

You have proven end-to-end inspection of inbound and outbound traffic for the Redwood Industries AWS workload.

### Architecture Review

![REFERENCE ARCHITECTURE](images/reference-architecture-lab3.png)

### Key Takeaways

1. **Address objects pay rent in policy clarity.** Defining `TESTVM-INTERNAL` once means every policy and VIP referencing the test VM is self-documenting. In a production environment with dozens of workloads, the difference between `10.100.2.10/32` and `TESTVM-INTERNAL` is the difference between a policy table you can read and one you can't.

2. **VIPs handle destination NAT, policies handle authorization.** A VIP alone won't let traffic through — FortiGate still requires an explicit firewall policy whose Destination references the VIP (or VIP group). This separation matters when a single VIP is used in multiple policies with different sources or services.

3. **NAT direction matches traffic direction.** The inbound `testvm_access_vip` policy does **not** enable source NAT — the VIP already handled destination NAT, and the test VM should see the real Internet source IP for accurate logging. The outbound `internet_access` policy **does** enable source NAT — without it, the test VM's private IP would never get a reply from the Internet.

4. **The Elastic IP is invisible inside FortiGate.** All FortiOS logs and tools show `port1`'s private VPC IP (`10.100.1.x`) as the NAT'd source. The Elastic IP substitution happens at the AWS IGW, one hop downstream. This is fundamentally different from Azure, where the public IP is presented directly on the FortiGate vNIC. When troubleshooting, always cross-check with `curl https://ifconfig.me` from inside the VPC to see the actual egress IP.

### Quick Reference

**Test VM access (from your workstation):**

| Service | Command |
| --- | --- |
| SSH | `ssh -i ~/.ssh/aws-101/Redwood-AWS-FGT-Key.pem -p 2222 ubuntu@<FortiGate-EIP>` |
| HTTP (once an HTTP server is running on the VM) | `curl http://<FortiGate-EIP>:8080` |

**Useful commands from inside the test VM:**

| Purpose | Command |
| --- | --- |
| Test ICMP egress | `ping -c 3 8.8.8.8` |
| Test DNS resolution | `ping -c 3 www.google.com` |
| Test HTTPS egress | `curl -I https://www.fortinet.com` |
| See your apparent public IP (should equal FortiGate EIP) | `curl -s https://ifconfig.me` |
| Show local IP and route | `ip addr show && ip route show` |

**FortiGate objects created:**

| Object | Type | Purpose |
| --- | --- | --- |
| `TESTVM-INTERNAL` | Address | Names the test VM in policies and VIPs |
| `TESTVM-INTERNAL-VIP-SSH` | Virtual IP | DNAT `:2222 → 10.100.2.10:22` |
| `TESTVM-INTERNAL-VIP-HTTP` | Virtual IP | DNAT `:8080 → 10.100.2.10:80` |
| `TESTVM-INTERNAL-VIPGRP` | VIP Group | Aggregates both VIPs for one policy |
| `testvm_access_vip` | Firewall Policy | Allow `port1 → port2` for VIP destinations |
| `internet_access` | Firewall Policy | Allow `port2 → port1`, NAT enabled |

### Next Steps

Ready for [***Lab 4 — Site-to-Site VPN Configuration***](/aws-101-lab4/README.md)

In Lab 4 you will:

- Configure an IPsec VPN tunnel between the AWS FortiGate (`Redwood-AWS-FGT`) and the on-premises FortiGate
- Establish bidirectional connectivity between the AWS `Private-Subnet` (`10.100.2.0/24`) and the on-prem network (`192.168.0.0/22`)
- Create FortiGate firewall policies for the VPN traffic
- Validate end-to-end ping and TCP traffic across the tunnel
- Inspect tunnel state and traffic in FortiGate's IPsec dashboard

This will complete the hybrid-cloud connectivity story for Redwood Industries.

---

## Configuration Checklist

Before moving to Lab 4, verify:

- [ ] `Redwood-AWS-TestVM` is **Running** with **2/2 checks passed**, in `Private-Subnet`, private IP `10.100.2.10`, **no public IP**
- [ ] Security group `Redwood-AWS-TestVM-SG` allows SSH (22) and HTTP (80)
- [ ] FortiGate address object `TESTVM-INTERNAL` exists at `10.100.2.10/32`
- [ ] VIPs `TESTVM-INTERNAL-VIP-SSH` (`:2222 → :22`) and `TESTVM-INTERNAL-VIP-HTTP` (`:8080 → :80`) exist on `port1`
- [ ] VIP group `TESTVM-INTERNAL-VIPGRP` aggregates both VIPs
- [ ] Firewall policy `testvm_access_vip` (port1 → port2, dest = VIPGRP, services SSH+HTTP) is enabled
- [ ] Firewall policy `internet_access` (port2 → port1, NAT enabled) is enabled
- [ ] SSH from your workstation reaches the test VM via `<FortiGate-EIP>:2222`
- [ ] From the test VM, `ping 8.8.8.8` and `curl -I https://www.fortinet.com` succeed
- [ ] `curl -s https://ifconfig.me` from the test VM returns `<FortiGate-Elastic-IP>`
- [ ] **Forward Traffic** log shows `internet_access` matches with `accept` action and visible NAT translation
- [ ] **FortiView Sources** lists `10.100.2.10` with realistic session counts

---

## Troubleshooting Guide

### SSH to the Test VM Fails

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Connection timed out` on TCP/2222 | `Redwood-AWS-FGT-SG` doesn't allow SSH from your IP, or your network blocks outbound 2222 | EC2 → Security Groups → `Redwood-AWS-FGT-SG` → add a custom TCP rule allowing port 2222 from your IP |
| `Connection refused` on TCP/2222 | VIP or firewall policy missing | Verify `TESTVM-INTERNAL-VIP-SSH` exists and is part of `TESTVM-INTERNAL-VIPGRP`; verify `testvm_access_vip` is **Enabled** |
| `Permission denied (publickey)` | Wrong key file or username | Use `ubuntu` (Ubuntu Server 26.04), `Redwood-AWS-FGT-Key.pem`, and `chmod 400` on the key file |
| Connects, then immediately drops | `Redwood-AWS-TestVM-SG` blocks SSH from FortiGate's `port2` IP | Verify the SG rule's source is `0.0.0.0/0` |

### Test VM Can SSH In But Can't Reach the Internet

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `ping 8.8.8.8` shows packets transmitted but 100% loss | NAT disabled on `internet_access` policy | Edit policy, ensure **NAT** is **Enabled** with **Use Outgoing Interface Address** |
| DNS resolution fails (`ping www.google.com`) | NAT works but DNS UDP/53 not reaching FortiGate | Verify `internet_access` policy Service is `ALL` (not just HTTP/HTTPS); verify `Redwood-AWS-RT-Private` default route is `0.0.0.0/0 → port2 ENI` |
| Internet works for some destinations, fails for others | FortiGuard / web filter blocking | Check **Log & Report → Forward Traffic** for `deny` entries; the workshop policy doesn't enable security profiles, so denies should be rare |
| `curl ifconfig.me` returns the wrong IP | Test VM has an auto-assigned public IP | EC2 → Instance details → Networking — primary ENI should show no public IPv4. If it does, you forgot to disable "Auto-assign public IPv4" at launch |

### No Logs Visible in Forward Traffic

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| No new entries despite test traffic | Policy logging set to "Security events" only | Edit each policy → set **Log allowed traffic** to **All sessions** |
| Logs appear but timestamps are wrong | FortiGate system clock drift | **System → Settings** — confirm time zone and NTP |
| Logs missing for outbound but present for inbound | `internet_access` policy disabled or in wrong order | Check **Status: Enabled**; check policy ordering — `internet_access` should not sit below an explicit deny rule |

### Policy Lookup Tool

When debugging which policy applies to a flow, use FortiGate's built-in lookup:

- **Policy & Objects > Firewall Policy > Policy Match**
- Incoming interface: `port2`
- Protocol: `ICMP`
- ICMP type: `Ping request`
- Source: `10.100.2.10`
- Authentication: `Any`
- Destination Address: `8.8.8.8`

  ![POLICY MATCH](images/step_final.png)

- Click **Find matching policy** — FortiGate shows the matching policy or "Implicit Deny" if no matching policy were found

  ![ACCEPT](images/step_final_2.png)

---

## FortiGate CLI Quick Reference

For attendees comfortable with the CLI, FortiGate exposes useful diagnostics:

```text
# Show all firewall policies
show firewall policy

# Show address objects and VIPs
show firewall address
show firewall vip
show firewall vipgrp

# Watch live sessions for the test VM
diagnose sys session filter src 10.100.2.10
diagnose sys session list

# Live packet capture on port2 (test VM IP)
diagnose sniffer packet port2 "host 10.100.2.10" 4 20

# Trace policy matching for a single flow
diagnose debug flow filter addr 10.100.2.10
diagnose debug flow trace start 20
diagnose debug enable
# (generate traffic from the test VM)
diagnose debug disable
diagnose debug reset
```

---

*Lab Guide Version 1.0 — May 2026*
*Questions? Ask your instructor or refer to the troubleshooting section.*
