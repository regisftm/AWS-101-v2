# AWS-101 Lab 2: FortiGate VM Deployment & Traffic Steering

## Lab Overview

### Objective

Deploy a FortiGate-VM from the AWS Marketplace into the VPC built in Lab 1, attach a second Elastic Network Interface (ENI) so the firewall has both an external (`port1`) and internal (`port2`) presence, license the appliance with FortiFlex, and configure VPC route tables to steer all subnet traffic through FortiGate for inspection.

This lab combines the appliance deployment with the routing configuration — by the end, every packet entering or leaving `Private-Subnet` will pass through FortiGate, and `Public-Subnet` will have a working path to the Internet via the IGW from Lab 1.

### What You'll Build

- A FortiGate-VM EC2 instance (`Redwood-AWS-FGT`) running FortiOS 8.0.x on a `c5.large` instance in `ca-central-1a`
- `port1` ENI in `Public-Subnet` with an Elastic IP for management and Internet egress
- `port2` ENI in `Private-Subnet` with a **static** private IP of `10.100.2.4` for traffic inspection
- **Source/Destination check** disabled on both ENIs (required for any AWS instance acting as a router/firewall)
- FortiFlex license activated, FortiGuard registered, FortiGate GUI reachable via HTTPS
- `Redwood-AWS-RT-Private` route table — `0.0.0.0/0` → FortiGate `port2` ENI, associated with `Private-Subnet` (the Public Subnet's route table was built in Lab 1)

### Architecture After Lab 2

![REFERENCE ARCHITECTURE](images/reference-architecture-lab2.png)

### Business Context

Redwood Industries' security policy requires every workload — on-premises **and** cloud — to be inspected by FortiGate before traffic reaches the Internet or another network segment. In Lab 1 you built the AWS networking foundation. In this lab, you deploy the security appliance that turns that foundation into an inspected enclave: a FortiGate-VM acting as the single point of egress and ingress for the `Private-Subnet`. Once the route tables are associated, every workload deployed in Lab 3 will have its traffic steered through FortiGate automatically — no per-VM configuration required.

---

## Prerequisites

- Lab 1 completed successfully (VPC, two subnets, IGW)
- IAM user/role with `AmazonEC2FullAccess` and `AmazonVPCFullAccess` (or equivalent inline policies for `ec2:*`, `vpc:*`, and Marketplace subscription rights)
- A **FortiFlex token** provided by your instructor (or your own active FortiFlex entitlement)
- A modern terminal capable of using an OpenSSH-format private key (`.pem`) on macOS / Linux / WSL, or PuTTY on Windows. You will create a workshop-dedicated EC2 **key pair** in Step 2 — there is no need to bring an existing one
- A modern browser (Chrome or Firefox) for the FortiGate GUI

> [!IMPORTANT]
> The FortiGate-VM AMI is a **paid Marketplace product**. Even with a BYOL listing (no software charge), you must accept the seller's End-User Licence Agreement once per AWS account before launch. Step 1 walks through this acceptance — it is a one-time action.

---

## PART 1: FortiGate VM Deployment

This part covers everything from accepting the AMI in AWS Marketplace through getting a licensed, reachable FortiGate GUI.

---

## Step 1: Subscribe to the FortiGate-VM AMI in AWS Marketplace

Before you can launch a FortiGate EC2 instance, you must subscribe to the BYOL Marketplace listing. This is a one-time per-account action that records the AWS Marketplace EULA acceptance.

1. **Open the AWS Marketplace:**
   - In the top search bar, type `Marketplace`
   - Click **AWS Marketplace**
   - Confirm you are still in **ca-central-1** (top-right region selector)

   ![MARKETPLACE](images/step1.1.png)

2. **Search for the FortiGate BYOL listing:**
   - Click **Discover products**
   - In the search bar, type `Fortinet FortiGate Next-Generation Firewall`
   - Press Enter

   ![DISCOVER](images/step1.2.png)

3. **Select the correct listing:**
   - Click the listing published by **Fortinet, Inc.** with NO **"(PAYG)"** in the title
   - Review the overview pane (optional)

   ![RIGHT PRODUCT](images/step1.3.png)

> [!NOTE]
> Fortinet publishes several FortiGate listings. The one you want is the BYOL (Bring Your Own License) variant — the PAYG (Pay-As-You-Go) listing bills the licence by the hour and is **not** what FortiFlex uses.

4. **Subscribe to the listing:**
   - Click **View purchase options**
   - Click **Subscribe**
   - Accept the EULA when prompted (subscription is granted immediately)

   ![PURCHASE](images/step1.4.png)

5. **Skip the in-Marketplace launch wizard:**
   - The next page offers a **Continue to Configuration → Launch** flow — **do not use it** (it provides limited control over networking)
   - Close this page and return to the EC2 console; you will launch the instance manually in Step 3 to retain full control over ENI placement and tagging

### Validation

- [x] **AWS Marketplace > Manage subscriptions** lists `Fortinet FortiGate VM Next-Generation Firewall` in the **Active subscriptions** tab
- [x] No EULA prompt remains outstanding
  
![VALIDATION](images/step1.validation.png)

---

## Step 2: Create the EC2 Key Pair

EC2 key pairs provide SSH access to Linux instances and serve as the initial credential factor for many AMIs. For this workshop, you will create a dedicated key pair so that all attendees follow the same naming convention and the key can be cleanly deleted at the end of the lab — even if you already have other key pairs in your account.

> [!IMPORTANT]
> AWS only allows you to download the **private** half of the key pair **once**, at the moment of creation. If the download is lost or the file is misplaced, you must delete the key pair and create a new one. AWS never stores the private key.

1. **Open the Key Pairs console:**
   - In the top search bar, type `EC2` and click **EC2** (the service result)
   - In the left navigation menu under **Network & Security**, click **Key Pairs**
   - Confirm the region selector at the top right shows **Canada (Central) ca-central-1**

   ![KEY PAIRS](images/step2.1.png)

2. **Create the key pair:**
   - Click **Create key pair** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `Redwood-AWS-FGT-Key` |
     | Key pair type | `RSA` |
     | Private key file format | `.pem` (macOS / Linux / WSL / OpenSSH) **or** `.ppk` (Windows / PuTTY) |
     | Tags | |
     | `Project` | `Redwood-AWS-101` |

   - Click **Create key pair**. Your browser will immediately download `Redwood-AWS-FGT-Key.pem` (or `.ppk`).

   ![CREATE KEY PAIR](images/step2.2.png)

3. **Move the file to a safe location and tighten permissions:**
   - On **macOS / Linux / WSL** (the `.pem` file must not be world-readable or SSH will refuse to use it):

     ```bash
     mkdir -p ~/.ssh/aws-101
     mv ~/Downloads/Redwood-AWS-FGT-Key.pem ~/.ssh/aws-101/
     chmod 400 ~/.ssh/aws-101/Redwood-AWS-FGT-Key.pem
     ```

   - On **Windows (PowerShell)** — keep the `.ppk` file with PuTTY's other keys (commonly `C:\Users\<you>\Documents\AWS-101\`). PuTTY does not enforce a permissions check.

4. **Verify the key pair exists in AWS:**
   - Return to the **Key Pairs** console
   - Confirm `Redwood-AWS-FGT-Key` is listed with **Type: rsa** and a **Fingerprint** value

   ![VERIFICATION](images/step2.4.png)

### Validation

- [x] `Redwood-AWS-FGT-Key` appears in **EC2 → Key Pairs** with type **rsa**
- [x] You have the downloaded `.pem` (or `.ppk`) file saved in a known, secure location
- [x] On macOS / Linux / WSL, `ls -l Redwood-AWS-FGT-Key.pem` shows permissions `-r--------` (400)

---

## Step 3: Launch the FortiGate EC2 Instance

In this step you launch a new EC2 instance from the FortiGate AMI, placing its primary network interface (`port1`) in `Public-Subnet`. The secondary interface (`port2`) is added in Step 5.

1. **Open the EC2 console:**
   - In the top search bar, type `EC2` and click **EC2** (the service result)
   - In the left navigation menu, click **Instances**
   - Click **Launch instances**

   ![LAUNCH INSTANCE](images/step3.1.png)

2. **Name and tags:**
   - Click **Add additional tags** and enter the values below. The `Name` tag is set automatically from the **Name** field at the top.

     | Parameter | Value |
     | --- | --- |
     | Name | `Redwood-AWS-FGT` |
     | Tags | | 
     | `Project` | `Redwood-AWS-101` |

   - Click **Add tag to resource types → Instances, Volumes, Network interfaces** so the tags propagate to the EBS volume and the primary ENI.

   ![TAGS](images/step3.2.png)

3. **Application and OS Images (Amazon Machine Image):**
   - Click **Browse more AMIs**
   - Click the **AWS Marketplace AMIs** tab
   - Search for `FortiGate BYOL`
   - Select the **Fortinet FortiGate (BYOL) Next-Generation Firewall** listing and click **Select**

   ![SELECT AMI](images/step3.3.png)

4. **Instance type:**
   - Use the parameter below.

     | Parameter | Value |
     | --- | --- |
     | Instance type | `c5.large` (2 vCPU, 4 GB, dedicated CPU, ENA) |

   ![INSTANCE TYPE](images/step3.4.png)

> [!NOTE]
> `c5.large` is the smallest officially supported FortiGate-VM instance type for AWS. For production workloads, Fortinet recommends `c5.xlarge` or larger. For this lab's traffic profile, `c5.large` is sufficient and minimizes EC2 cost.

5. **Key pair (login):**
   - In the **Key pair name — required** dropdown, select `Redwood-AWS-FGT-Key` (created in Step 2)
   - Do **not** click **Create new key pair** — using the workshop key pair keeps tagging and clean-up consistent

6. **Network settings:**
   - Click **Edit** to expand the panel and use the parameters below. This is the most error-prone screen — match every value carefully.

     | Parameter | Value |
     | --- | --- |
     | VPC | `Redwood-AWS-VPC` |
     | Subnet | `Public-Subnet` |
     | Auto-assign public IP | **Disable** |
     | Firewall (security groups) | **Create security group** |
     | Security group name | `Redwood-AWS-FGT-SG` |
     | Description | `Management and inspection access for Redwood-AWS-FGT` |

     ![NETWORK SETTINGS](images/step3.6.a.png)

   - In the **Inbound security group rules** section, replace the default rule with the three rules below.

     | Type | Protocol | Port range | Source type | Source | Description |
     | --- | --- | --- | --- | --- | --- |
     | ssh | TCP | 22 | My IP | (auto-filled) | FortiGate CLI |
     | HTTPS | TCP | 443 | My IP | (auto-filled) | FortiGate GUI |
     | Custom TCP | TCP | 2222 | Anywhere | `0.0.0.0/0` | Incoming access to server |
     | Custom TCP | TCP | 8080 | Anywhere | `0.0.0.0/0` | Incoming access to server |
     | All traffic | All | All | Custom | 10.100.0.0/16 | Traffic from VPC |

     ![SG GROUP CONFIG I](images/step3.6.b.gif)
     ![SG GROUP CONFIG II](images/step3.6.d.png)

> [!IMPORTANT]
> The "Auto-assign public IP" must be **Disabled**. You will associate a dedicated **Elastic IP** with `port1` in Step 4 — an auto-assigned public IP would be released the first time the instance is stopped, which would invalidate the FortiFlex licence binding.

7. **Configure storage:**
   - Keep the default `2 GiB` boot volume and add a second volume for FortiGate logs, using the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Volume 1 (root) | `2 GiB`, `gp3` |
     | Volume 2 (log) | `30 GiB`, `gp3`, device name `/dev/sdb` |

8. **Review and launch:**
   - On to the **Summary** panel on the right
   - Confirm **Number of instances** is `1`
   - Click **Launch instance**

   ![LAUNCH INSTANCE](images/step3.8.png)

9. **Wait for the instance to reach `Running` state:**
    - Click **View all instances**
    - Refresh the **Instances** page until the **Instance state** shows **Running**
    - Wait until the **Status check** column shows **3/3 checks passed** (typically 2–4 minutes for a `c5.large`)
    - Click on the reload button to update the **Status check**

   ![RUNNING](images/step3.9.png)

### Validation

- [x] `Redwood-AWS-FGT` appears in the **Instances** list with state **Running**
- [x] Instance type shows `c5.large`
- [x] AMI ID begins with `ami-` and the AMI name contains `FortiGate` and `8.0`
- [x] **Networking** tab shows the primary ENI is in `Public-Subnet` with **no public IPv4 address yet**
- [x] Security group `Redwood-AWS-FGT-SG` is attached and contains the five inbound rules above

---

## Step 4: Allocate an Elastic IP and Associate It with `port1`

In AWS, public IPv4 addresses come in two flavours: **auto-assigned** (released when the instance stops) and **Elastic IP** (persistent, account-bound). FortiGate's licence is bound to its public IP, so a stable Elastic IP is required.

1. **Allocate the Elastic IP:**
   - In the EC2 console left navigation menu under **Network & Security**, click **Elastic IPs**
   - Click **Allocate Elastic IP address**
  
     ![ALLOCATE EIP](images/step4.1.a.png)

   - Use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Public IPv4 address pool | **Amazon's pool of IPv4 addresses** |
     | Network Border Group | `ca-central-1` |
     | Tags | |
     | `Name` | `Redwood-AWS-FGT-EIP` |
     | `Project` | `Redwood-AWS-101` |

   - Click **Allocate**

     ![ALLOCATE](images/step4.1.b.png)

2. **Associate the EIP with the FortiGate primary ENI:**
   - In the **Elastic IPs** list, select the row for the new EIP
   - Click **Actions → Associate Elastic IP address**

     ![ASSOCIATE EIP](images/step4.2.a.png)

   - Use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Resource type | **Network interface** |
     | Network interface | (primary ENI of `Redwood-AWS-FGT` — its description begins `Primary network interface`) |
     | Private IP address | (the only IP listed; a `10.100.1.x` address) |
     | Allow this Elastic IP address to be reassociated | **Unchecked** |

   - Click **Associate**

     ![ASSOCIATE](images/step4.2.b.png)

3. **Note the Elastic IP value:**
   - Copy the **Allocated IPv4 address** value (e.g., `15.222.x.x`)
   - You will use it to access the FortiGate GUI in Step 8 — record it somewhere safe

### Validation

- [x] The Elastic IP shows **Associated** with the FortiGate primary ENI
- [x] In **EC2 → Instances → Redwood-AWS-FGT → Details**, the **Public IPv4 address** field now shows the Elastic IP value
- [x] You have written down the Elastic IP address (e.g., `15.222.x.x`)

---

## Step 5: Create the `port2` Elastic Network Interface

FortiGate's internal interface must live in `Private-Subnet` and must use the **static** private IP `10.100.2.4` so that the Lab 2 route table can use it as a deterministic next-hop. You will create this ENI as a separate object first, then attach it in Step 6.

1. **Open the Network Interfaces console:**
   - In the EC2 console left navigation menu under **Network & Security**, click **Network Interfaces**
   - Click **Create network interface**

     ![CREATE ENI](images/step5.1.png)

2. **Configure the ENI:**
   - Use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Description | `Redwood-AWS-FGT port2 (internal inspection interface)` |
     | Subnet | `Private-Subnet` |
     | Interface type | **ENA** |
     | Private IPv4 address — assignment | **Custom** |
     | Private IPv4 address | `10.100.2.4` |
     | Security groups | `Redwood-AWS-FGT-SG` |
     | Tags |
     | `Name` | `Redwood-AWS-FGT-port2` |
     | `Project` | `Redwood-AWS-101` |

   - Click **Create network interface**

     ![CREATE](images/step5.2.gif)

### Validation

- [x] `Redwood-AWS-FGT-port2` appears in **Network Interfaces** with **Status: Available**
- [x] Subnet shows `Private-Subnet`, primary private IPv4 shows `10.100.2.4`
- [x] **Source/destination check** column shows **Enabled** (you will disable it in Step 7)

---

## Step 6: Attach the `port2` ENI to the FortiGate Instance

The new ENI exists but is unattached. Attaching it adds a second virtual NIC to the running FortiGate instance, which FortiOS will detect on its next boot or interface scan.

1. **Stop the FortiGate instance:**
   - In the EC2 console left navigation menu, click **Instances**
   - Select the row for `Redwood-AWS-FGT`
   - Click **Instance state → Stop instance** and confirm

     ![STOP INSTANCE](images/step6.1.png)

   - Wait until the **Instance state** column shows **Stopped**

> [!NOTE]
> Attaching an additional ENI to a running instance is technically supported, but FortiOS sometimes does not detect the new interface until the next reboot. Stopping the instance first guarantees clean detection.

2. **Attach the ENI:**
   - With `Redwood-AWS-FGT` still selected, click **Actions → Networking → Attach network interface** and use the parameters below.

     ![ATTACH ENI](images/step6.2.a.png)

     | Parameter | Value |
     | --- | --- |
     | VPC | Redwood-AWS-VPC |
     | Network interface | (`Redwood-AWS-FGT port2` (`internal inspection interface`)) |

   - Click **Attach**

     ![ATTACH](images/step6.2.b.png)

3. **Verify the attachment:**
   - With `Redwood-AWS-FGT` still selected, click the **Networking** tab in the lower details pane
   - Confirm two **Network interfaces** are listed
   - Confirm the primary (`port1`) is in `Public-Subnet` and the secondary (`port2`) is in `Private-Subnet` at `10.100.2.4`

   ![VERIFY](images/step6.3.gif)

### Validation

- [x] `Redwood-AWS-FGT` shows two attached network interfaces
- [x] `port1` ENI is in `Public-Subnet` with the Elastic IP from Step 4 associated
- [x] `port2` ENI is in `Private-Subnet` with private IP `10.100.2.4`

---

## Step 7: Disable Source/Destination Check on Both ENIs

By default, AWS drops any packet that arrives at an ENI whose source or destination IP does not match that ENI's IP. This is fine for normal application servers but **breaks** any instance that needs to forward traffic — including FortiGate. You must disable this check on both ENIs.

1. **Disable on `port1` (primary ENI):**
   - In the EC2 console left navigation menu under **Network & Security**, click **Network Interfaces**
   - Select the primary ENI of `Redwood-AWS-FGT` (the one in `Public-Subnet`)
   - Click **Actions → Change source/dest. check** and use the parameter below.

     | Parameter | Value |
     | --- | --- |
     | Source/destination checking | **Unchecked** |

   - Click **Save**

     ![UNCHECK SRC/DST CHECKING](images/step7.1.gif)

2. **Disable on `port2`:**
   - In the same **Network Interfaces** list, select `Redwood-AWS-FGT-port2`
   - Click **Actions → Change source/dest. check**
   - Set **Source/destination checking** to **Unchecked** and click **Save**

3. **Start the FortiGate instance:**
   - In the EC2 console left navigation menu, click **Instances**
   - Select the row for `Redwood-AWS-FGT`
   - Click **Instance state → Start instance**
   - Wait until the **Status check** column shows **3/3 checks passed**

     ![START INSTANCE](images/step7.3.png)

### Validation

- [x] Both ENIs show **Source/dest. check: Disabled** in their **Details** tab
- [x] `Redwood-AWS-FGT` is back to **Running** with **3/3 checks passed**
- [x] The Elastic IP is still associated with `port1` (verify in **EC2 → Elastic IPs**)

---

## Step 8: Activate the FortiFlex Licence and Access the FortiGate GUI

FortiGate-VM has a permanent **evaluation** VM license limited to 1 CPUs and 2 GB RAM. Before it can route traffic for Lab 3, you must register it against your FortiFlex entitlement.

1. **Open the FortiGate GUI:**
   - In your browser, navigate to `https://<Elastic-IP-from-Step-4>` (note the **HTTPS** prefix — port 443 is used)
   - You will see a "Your connection is not private" warning because FortiGate ships with a self-signed certificate — this is expected
   - **Chrome:** click **Advanced** → **Proceed to (IP) (unsafe)**
   - **Firefox:** click **Advanced** → **Accept the Risk and Continue**

2. **First-time login:**
   - Use the credentials below.

     | Parameter | Value |
     | --- | --- |
     | Username | `admin` |
     | Password | (the EC2 instance ID — e.g., `i-0abc123def4567890`) |

   > [!NOTE]
   > Unlike many appliances, the AWS FortiGate-VM uses the **EC2 instance ID** as the initial admin password. Find it in **EC2 → Instances → Redwood-AWS-FGT → Details → Instance ID**.

3. **Set a new admin password:**
   - When prompted, enter a strong password (12+ characters, mixed case, digits, symbol)
   - Confirm the password
   - Record it securely — it is not recoverable

4. **Activate FortiFlex:**
   - In the **VM Licence** dialog, select **Activation type: FortiFlex token** and use the parameter below.

     | Parameter | Value |
     | --- | --- |
     | FortiFlex token | (the FortiFlex token provided by your instructor) |

   - Click **OK** and then confirm the system reboot. FortiGate will contact the FortiCare service via its `port1` Internet path — activation typically completes in 30–60 seconds

   ![FORTIFLEX](images/step8.4.png)

5. **Verify the licence:**
   - After the reboot, login to FortiGate again.
   - Go through the FortiGate Setup process.
   - In the FortiGate GUI left navigation menu, click **System → FortiGuard** (or use the **Dashboard → Status → Licence widget**)
   - Confirm **VM licence**: `Valid`
   - Confirm **Support contract**: `Valid`
   - Confirm **IPS, Advanced Malware Protection, URL, DNS & Video Filtering**: all show `Licensed` and `Up to date`

   ![LICENSES](images/step8.5.png)

6. **Confirm interface state:**
   - In the FortiGate GUI left navigation menu, click **Network → Interfaces**
   - Confirm the interfaces match the values below.

     | Interface | Expected state |
     | --- | --- |
     | `port1` | `Up`, IP `10.100.1.x/24` (DHCP from VPC) |
     | `port2` | `Up`, IP `10.100.2.4/24` (DHCP from VPC, resolves to the static reservation) |

     ![INTERFACES](images/step8.6.a.png)

   - `port2` may not be configured correctly. If this is the case, configure a static IP address of `10.100.2.4/24` on port2

     ![CONFIG PORT2](images/step8.6.b.png)

> [!TIP]
> AWS does not surface the Elastic IP inside the FortiGate OS — `port1` will show the **private** `10.100.1.x` address. The public Elastic IP is provided by the IGW via 1:1 NAT, transparent to the appliance.

### Validation

- [x] You can log into the FortiGate GUI at `https://<Elastic-IP>`
- [x] Admin password has been changed from the default and recorded securely
- [x] FortiFlex licence shows **Valid** under **System → FortiGuard**
- [x] Both `port1` and `port2` interfaces show state **Up** in **Network → Interfaces**
- [x] `port2` IP is exactly `10.100.2.4`

---

## PART 2: Traffic Steering Configuration

`Public-Subnet` is already routed to the Internet — `Redwood-AWS-RT-Public` was created in Lab 1 and is what allowed Step 8 to reach the FortiGate GUI and activate the FortiFlex licence. The remaining piece is the **Private Subnet route table**, which can only be built now that FortiGate's `port2` ENI exists to serve as the next-hop.

`Redwood-AWS-RT-Private` will send `0.0.0.0/0` from `Private-Subnet` to FortiGate's `port2` ENI, so any workload deployed in Lab 3 has its egress automatically inspected by FortiGate. Without this route, an instance in `Private-Subnet` would have no path off the subnet at all (the VPC main route table only has the implicit `local` route).

---

## Step 9: Create and Configure the Private Subnet Route Table

This is the route table that turns FortiGate into the inspection point for `Private-Subnet`. The next-hop is FortiGate's `port2` ENI (not its IP, not the instance ID — AWS routes to the ENI object directly).

1. **Create the route table:**
   - In the VPC console left navigation menu under **Virtual private cloud**, click **Route tables**

     ![ROUTE TABLES](images/step9.1.png)

   - Click **Create route table** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Name | `Redwood-AWS-RT-Private` |
     | VPC | `Redwood-AWS-VPC` |
     | Tags | |
     | `Project` | `Redwood-AWS-101` |

   - Click **Create route table**

     ![CREATE ROUTE TABLE](images/step9.1.b.png)

2. **Add the default route to FortiGate `port2`:**
   - Open the new `Redwood-AWS-RT-Private` route table
   - Click the **Routes** tab
   - Click **Edit routes → Add route** and use the parameters below.

     | Parameter | Value |
     | --- | --- |
     | Destination | `0.0.0.0/0` |
     | Target | **Network Interface → `Redwood-AWS-FGT-port2`** (the ENI created in Step 5 — its IP is `10.100.2.4`) |

   - Click **Save changes**

   ![ADD ROUTE](images/step9.2.png)

> [!IMPORTANT]
> The target is the **ENI**, not an IP address or instance. AWS console will show the ENI's interface ID (`eni-xxxxxxxxxxxxxxxxx`) and its private IP `10.100.2.4`. If you select an IP target instead, the route will resolve incorrectly when the instance is replaced.

3. **Associate the route table with `Private-Subnet`:**
   - Click the **Subnet associations** tab
   - Click **Edit subnet associations** and use the parameter below.

     | Parameter | Value |
     | --- | --- |
     | Subnets to associate | `Private-Subnet` (`10.100.2.0/24`) |

   - Click **Save associations**

   ![SUBNET ASSOCIATION](images/step9.3.gif)

### Validation

- [x] `Redwood-AWS-RT-Private` exists with two routes: `10.100.0.0/16 → local` and `0.0.0.0/0 → eni-xxxxxxxx (Redwood-AWS-FGT-port2)`
- [x] `Private-Subnet` appears under **Subnet associations** for this route table
- [x] **VPC → Route tables** shows the **Main** route table is no longer associated with either subnet (both are now using their dedicated tables)

---

## Lab 2 Complete

You have now deployed and configured the security appliance for Redwood Industries' AWS environment.

### Architecture Review

Current state after Lab 2:

![REFERENCE ARCHITECTURE LAB 2](images/reference-architecture-lab2.png)

### Key Takeaways

1. **Two ENIs, two roles.** A FortiGate-VM on AWS always needs at least two ENIs in two subnets — one for the Internet-facing role and one for the inspection role. The primary ENI is bound to the instance at launch; secondary ENIs are created and attached as separate operations.

2. **Source/destination check is the AWS equivalent of "promiscuous mode".** Every ENI that forwards traffic on behalf of another instance must have this disabled. Forgetting to do so is the single most common reason FortiGate appears to be working but no inspected traffic flows through it.

3. **Routes target ENIs, not IPs.** The `0.0.0.0/0 → ENI` entry in `Redwood-AWS-RT-Private` references the `port2` ENI's interface ID, not the IP `10.100.2.4`. This is why the IP must be set as **static** when the ENI is created — replacing the instance later (e.g., during a resize or AMI update) preserves the ENI binding only if the IP is stable.

4. **The Elastic IP binds the FortiFlex licence.** Releasing or reassociating the EIP after activation can invalidate the licence binding. Treat `Redwood-AWS-FGT-EIP` as a long-lived resource for the lifetime of the lab.

5. **Both subnets now have explicit route tables.** Neither uses the VPC's Main route table any more. This is good practice — the Main table should remain empty (only `local`) so any forgotten/unassociated subnet defaults to a non-routable state rather than accidentally inheriting a permissive route.

### Quick Reference

**FortiGate access:**

| Property | Value |
| --- | --- |
| Management URL | `https://<Elastic-IP>` |
| Admin username | `admin` |
| Admin password | (the password you set during first login) |
| SSH | `ssh -i ~/.ssh/aws-101/Redwood-AWS-FGT-Key.pem admin@<Elastic-IP>` |

**Network configuration:**

| Interface | Subnet | Private IP | Notes |
| --- | --- | --- | --- |
| `port1` | `Public-Subnet` | DHCP `10.100.1.x` | Elastic IP `15.x.x.x` for management & egress |
| `port2` | `Private-Subnet` | `10.100.2.4` (static) | Next-hop for `Private-Subnet` default route |

**Route tables:**

| Route table | Default route target | Associated subnet |
| --- | --- | --- |
| `Redwood-AWS-RT-Public` | `Redwood-AWS-IGW` | `Public-Subnet` |
| `Redwood-AWS-RT-Private` | `Redwood-AWS-FGT-port2` ENI | `Private-Subnet` |

### Next Steps

Ready for [***Lab 3 — Security Policies & Traffic Testing***](/aws-101-lab3/README.md)

In Lab 3 you will:

- Deploy a test workload (Ubuntu Server 26.04 LTS) in `Private-Subnet`
- Create FortiGate firewall policies to allow inspected outbound Internet access (with NAT)
- Create a Virtual IP (VIP) and policy to expose SSH on the test workload through FortiGate's Elastic IP
- Generate test traffic and watch it appear in **FortiView → Sources** and **Log & Report → Forward Traffic**
- Validate that egress is now NAT'd to the FortiGate Elastic IP rather than the workload's private IP

---

## Configuration Checklist

Before moving to Lab 3, verify:

- [ ] `Redwood-AWS-FGT-Key` key pair exists in **EC2 → Key Pairs**, and the private key file is saved locally with permissions `400`
- [ ] `Redwood-AWS-FGT` is **Running** with **3/3 checks passed**
- [ ] Two ENIs attached: `port1` in `Public-Subnet`, `port2` in `Private-Subnet`
- [ ] `port2` private IP is exactly `10.100.2.4`
- [ ] **Source/destination check** is **Disabled** on **both** ENIs
- [ ] Elastic IP `Redwood-AWS-FGT-EIP` is associated with the `port1` ENI
- [ ] FortiGate GUI reachable at `https://<Elastic-IP>`
- [ ] FortiFlex licence shows **Valid** in **System → FortiGuard**
- [ ] `Redwood-AWS-RT-Public` exists with `0.0.0.0/0 → Redwood-AWS-IGW`, associated with `Public-Subnet`
- [ ] `Redwood-AWS-RT-Private` exists with `0.0.0.0/0 → Redwood-AWS-FGT-port2` ENI, associated with `Private-Subnet`
- [ ] All resources tagged with `Project=Redwood-AWS-101`

### AWS CLI Verification (Optional)

For advanced users with the AWS CLI v2 configured for `ca-central-1`:

```bash
export AWS_DEFAULT_REGION=ca-central-1

# Verify FortiGate instance state and instance type
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Redwood-AWS-FGT" \
  --query "Reservations[0].Instances[0].{InstanceId:InstanceId,State:State.Name,Type:InstanceType,PublicIp:PublicIpAddress}" \
  --output table

# Verify both ENIs and their source/dest check status
aws ec2 describe-network-interfaces \
  --filters "Name=tag:Project,Values=Redwood-AWS-101" \
  --query "NetworkInterfaces[].{ENI:NetworkInterfaceId,Subnet:SubnetId,PrivateIp:PrivateIpAddress,SrcDstCheck:SourceDestCheck,Status:Status}" \
  --output table

# Verify both route tables and their associations
aws ec2 describe-route-tables \
  --filters "Name=tag:Project,Values=Redwood-AWS-101" \
  --query "RouteTables[].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].{Dest:DestinationCidrBlock,Target:GatewayId||NetworkInterfaceId},AssocSubnets:Associations[].SubnetId}" \
  --output table
```

Expected: `SrcDstCheck` is `false` on both ENIs, both route tables list one subnet association each, and the `0.0.0.0/0` targets are the IGW (`igw-...`) and the `port2` ENI (`eni-...`) respectively.

---

## Troubleshooting Guide

### Cannot Reach the FortiGate GUI

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Browser hangs / times out | Security group missing HTTPS rule from your IP | **EC2 → Security Groups → Redwood-AWS-FGT-SG → Edit inbound rules** — add HTTPS (443) from **My IP** |
| `ERR_CERT_AUTHORITY_INVALID` | Self-signed FortiGate certificate | Expected — click **Advanced → Proceed** in the browser |
| Connection refused | Instance still booting, or `port1` not yet up | Wait 2–3 minutes after instance reaches **Running**; refresh the EC2 status check |
| Page loads but login fails | Wrong default password — you used `admin` / `password` instead of `admin` / `<instance-id>` | Default password on AWS is the **EC2 instance ID** |
| Page loads from one network but not another | Source IP changed (e.g., VPN on/off) and the SG rule is bound to your old IP | Update the SG rule **Source** to your current IP |

### `port2` IP Is Not `10.100.2.4`

- Symptom: ENI has a different `10.100.2.x` IP, or DHCP assigned a different address
- Cause: `port2` ENI was created without specifying a custom private IP, or the static IP was not selected
- Fix: Delete the ENI and recreate with **Private IPv4 address — Custom → 10.100.2.4** (Step 5)

### FortiFlex Licence Activation Fails

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| "Failed to contact FortiCare" | FortiGate cannot reach Internet via `port1` | Confirm Lab 1 Step 6 is complete (`Redwood-AWS-RT-Public` associated with `Public-Subnet`); test from FortiGate CLI: `execute ping fortiguard.com` |
| "Token already used" | Token previously consumed by another VM | Request a fresh token from your instructor |
| "License invalid, please contact FortiCare" | Token format wrong, or token region-restricted | Verify the token string was pasted without leading/trailing whitespace; verify with the instructor that the token is valid for FortiGate-VM (not FortiAnalyzer/Manager) |
| Licence stays in "Validating..." | Slow FortiCare response | Wait 5 minutes and refresh; if still stuck, navigate to **System → FortiGuard → Update FortiGuard servers** to retry |

### Test Traffic Doesn't Reach FortiGate

This becomes relevant in Lab 3, but check now:

- `Redwood-AWS-RT-Private` shows the `0.0.0.0/0 → eni-xxx` route (not `0.0.0.0/0 → instance i-xxx`)
- Both ENIs have **Source/dest. check: Disabled** (Step 7)
- `Private-Subnet` is associated with `Redwood-AWS-RT-Private` (not the VPC main route table)
- FortiGate firewall policies in Lab 3 are configured to allow `port2 → port1` with NAT enabled

### Instance Won't Start After ENI Attachment

- Symptom: After attaching `port2` and starting the instance, it stays in **Pending** for >5 minutes or fails the status check
- Cause: ENI attached to wrong device index (must be `1`, not `0`)
- Fix: Stop instance, detach the secondary ENI, re-attach with **Device index = 1**

---

## Additional Resources

**AWS Documentation:**

- Elastic Network Interfaces: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html>
- Elastic IP Addresses: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html>
- VPC Route Tables: <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html>
- Disable source/destination check: <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck>

**Fortinet Documentation:**

- FortiGate-VM AWS Administration Guide: <https://docs.fortinet.com/document/fortigate-public-cloud/8.0.0/aws-administration-guide>
- Deploying FortiGate-VM on AWS: <https://docs.fortinet.com/document/fortigate-public-cloud/8.0.0/aws-administration-guide/403036/deploying-fortigate-vm-on-aws>
- BYOL licensing on AWS: <https://docs.fortinet.com/document/fortigate-public-cloud/8.0.0/aws-administration-guide/421030/byol>
- FortiFlex token activation: <https://docs.fortinet.com/document/fortiflex/latest>

---

*Lab Guide Version 1.0 — May 2026*
*Questions? Ask your instructor or refer to the troubleshooting section.*
