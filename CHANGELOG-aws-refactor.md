# CHANGELOG — AWS-only refactor (branch `aws-only-refactor`)

Lab Guide Version 1.0 (May 2026) → 1.1 (September 2026)

This refactor removes all cross-cloud content and makes AWS-101 a standalone AWS-native workshop. The module order, objectives, and lab outcomes are unchanged. It also applies one AWS naming and tagging convention, fixes technical inaccuracies, and adds concise AWS Well-Architected callouts where they apply to the single-FortiGate architecture.

---

## Naming and tagging

**Convention:** `<company>-<workshop>-<environment>-<resource>`. Zonal resources (subnets) also get the Availability Zone suffix. The convention is explained once, in Lab 1 → *Naming & Tagging Strategy*.

**Standard tags on every AWS resource:** `Name`, `Project=Redwood-AWS-101` (unchanged), `Environment=lab`, `Owner=<your-name>`. Every tag table in Labs 1–3 now includes `Environment` and `Owner`.

| Old name | New name |
| --- | --- |
| `Redwood-AWS-RG` | `redwood-aws101-lab-rg` |
| `Redwood-AWS-VPC` | `redwood-aws101-lab-vpc` |
| `Public-Subnet` | `redwood-aws101-lab-subnet-public-1a` |
| `Private-Subnet` | `redwood-aws101-lab-subnet-private-1a` |
| `Redwood-AWS-IGW` | `redwood-aws101-lab-igw` |
| `Redwood-AWS-RT-Public` | `redwood-aws101-lab-rt-public` |
| `Redwood-AWS-RT-Private` | `redwood-aws101-lab-rt-private` |
| `Redwood-AWS-FGT` | `redwood-aws101-lab-fgt` |
| `Redwood-AWS-FGT-SG` | `redwood-aws101-lab-fgt-sg` |
| `Redwood-AWS-FGT-Key` (and `.pem` / `.ppk` file) | `redwood-aws101-lab-kp` |
| `Redwood-AWS-FGT-EIP` | `redwood-aws101-lab-fgt-eip` |
| `Redwood-AWS-FGT-port2` | `redwood-aws101-lab-fgt-eni-port2` |
| `Redwood-AWS-TestVM` | `redwood-aws101-lab-testvm` |
| `Redwood-AWS-TestVM-SG` | `redwood-aws101-lab-testvm-sg` |

FortiOS object names (`TESTVM-INTERNAL*`, `testvm_access_vip`, `internet_access`, `to_aws`, `to_on_prem`) are unchanged.

---

## Changes per file

### `README.md` (workshop overview)

- Corrected the value statement. It no longer presents Gateway Load Balancer as a competing product (GWLB hosts FortiGate). The statement now positions FortiGate IPsec against managed Site-to-Site VPN connections and AWS Network Firewall for this single-VPC design.
- "bootcamp" → "workshop".

### `aws-101-lab1/README.md` (AWS Infrastructure Foundation)

- **Cross-cloud content removed:** the Resource Group comparison in the tagging intro, the VPC comparison, the "subnets are zonal" comparison, the whole 3-subnet comparison `<details>` block (now "Why isn't there a separate Protected subnet?", AWS-only), the intra-subnet routing contrast, and the NOTE on another provider's default outbound access. That NOTE is replaced with a one-line AWS explanation of explicit Internet egress.
- "Tagging Strategy" rewritten as **Naming & Tagging Strategy** (the convention plus 4 standard tags).
- Resource Group creation now tags the group with all 4 standard tags.
- Added **Well-Architected – Security** (sandbox account, MFA, no root) and **Well-Architected – Cost Optimization** (billable items; points to Clean-Up) callouts to the prerequisites.
- Follow-on course references aligned with Lab 4: AWS-102 = HA; AWS-103 = GWLB / Transit Gateway / east-west.
- Replaced the generic `alt text` label on `step6.3.a.png` with a descriptive one.
- Fortinet documentation links aligned to the same version as Lab 2.

### `aws-101-lab2/README.md` (FortiGate EC2 Deployment & Traffic Steering)

- IAM prerequisite fixed: `vpc:*` is not an IAM namespace, because VPC actions are `ec2:*`. Now lists the specific Marketplace permissions.
- The BYOL listing note now asks for the **x86_64** listing, to match `c5.large`.
- Softened the unverified claim about the smallest supported instance and Fortinet's recommended size. Added a **Well-Architected – Performance Efficiency** callout.
- Security group step: "three rules" → "five rules" (to match the table), and `ssh` → `SSH`.
- **Accuracy:** removed the claim that the FortiFlex licence is bound to the public IP / Elastic IP. It appeared in Steps 3 and 4 and in Key Takeaway 4. The guide now explains that the EIP is needed as a stable address for management, the VIPs, and the VPN peer.
- Added a **Well-Architected – Security** callout: separate security groups per ENI role in production, and restricted management.
- Storage step: check for the AMI's existing log volume instead of blindly adding a duplicate.
- Added a **Well-Architected – Security** callout: enforce IMDSv2 (Metadata version = V2 only).
- Step 9 route target: corrected the non-existent "IP target" wording. The guide now explains why to target the ENI rather than the instance.
- Key Takeaways: replaced the "promiscuous mode" analogy with an accurate description of source/destination check. Rewrote the ENI/static-IP reasoning.
- Added a **Well-Architected – Reliability** callout: a single FortiGate in one AZ is a SPOF, and FGCP A-P across AZs or GWLB is the production pattern.
- "On to the Summary panel" → "Go to the Summary panel".

### `aws-101-lab3/README.md` (Security Policies & Traffic Testing)

- Test VM instance type `t2.micro` (previous generation) → `t3.micro`.
- Test VM security group description now matches its rules. Added a **Well-Architected – Security** callout explaining why the source is `0.0.0.0/0` and how defence in depth still holds.
- Added an IMDSv2 **Well-Architected – Security** callout for the test VM.
- Fixed the shell prompt to the Ubuntu format `ubuntu@ip-10-100-2-10:~$`.
- SSH tip: the FortiGate SG must allow TCP/2222 (not "SSH from your IP"). Fixed the `0.0.0.0/16` typo.
- Reworded the IGW behaviour behind the NAT requirement. The IGW translates only private IPs that have an associated public IP; it has no separate "source validation check".
- Removed the cross-cloud comparison from Key Takeaway 4.
- Added a **Well-Architected – Operational Excellence** callout: send logs off-box.

### `aws-101-lab4/README.md` (Site-to-Site VPN)

- Added a **Well-Architected – Cost Optimization** callout to the business context.
- **NAT-T deep dive corrected:**
  - ESP does not "sign" the outer IP header. The real issue is that port-based NAT can't track ESP.
  - The NAT detection explanation now matches the traffic direction.
  - The "without NAT-T" failure mode is now framed around NAT detection and the lab's security group. The earlier claim that the IGW can't pass ESP was removed.
  - NAT-T keepalive and DPD are now separate concepts.
- Pre-lab parameter "Keepalive (DPD) frequency" → "NAT-T keepalive frequency".
- Made the IPsec dashboard path consistent on both FortiGates.
- Troubleshooting now points to "(Step 5)" instead of "(Step 6)".
- Test section headings no longer use the ambiguous "Redwood-AWS".
- Quick reference: the on-prem Windows host now uses `Test-NetConnection`, and the ports match the Step 6 tests.
- Checklist: removed the "This site is behind NAT" item, which was never configured.
- Roadmap: AWS-102 = FGCP active-passive across AZs (no longer "with NLB"). The east-west answer now points to AWS-103.
- **Clean-up rewritten** in dependency order:
  1. Instances
  2. EIP
  3. ENI
  4. Security groups
  5. Key pair and local key file
  6. VPC
  7. Resource Group
  8. Tag Editor sweep for leftovers

  Fixed the "Go to VPN" typo. Added a **Well-Architected – Cost Optimization / Sustainability** callout.
- Tunnel flapping troubleshooting row now points to the NAT-T keepalive.

---

## Images

### Removed

None. No image in the repository showed another cloud provider. All 5 architecture diagrams are AWS-native (AWS Cloud → Region → VPC → Availability Zone → subnets, AWS Architecture Icons).

### Image placeholders added

None. The refactor added no new visual steps (Well-Architected content is callouts only).

### Screenshots to retake (kept in place; they probably show old resource names or only the `Project` tag)

Verify each one during the retake. Update the names per the rename table, and show all 4 tags wherever a tag table is visible.

| Lab | Images |
| --- | --- |
| Lab 1 | `step1.3.b.png`, `step1.3.c.png`, `step1.4.png`, `step2.3.png`, `step2.4.png`, `step2.6.png`, `step3.2.png`, `step3.3.png`, `step4.3.png`, `step5.2.png`, `step5.3.png`, `step5.3.b.png`, `step6.1.b.png`, `step6.2.a.png`, `step6.2.b.png`, `step6.3.a.png`, `step6.3.b.png` |
| Lab 2 | `step2.2.png`, `step2.4.png`, `step3.2.png`, `step3.6.a.png`, `step3.6.b.gif`, `step3.6.d.png`, `step3.8.png`, `step3.9.png`, `step4.1.b.png`, `step4.2.a.png`, `step4.2.b.png`, `step5.2.gif`, `step6.1.png`, `step6.2.a.png`, `step6.2.b.png`, `step6.3.gif`, `step7.1.gif`, `step7.3.png`, `step9.1.b.png`, `step9.2.png`, `step9.3.gif`, `reference-architecture-lab2.png` (instance label) |
| Lab 3 | `step1.2.png`, `step1.6.a.png`, `step1.6.b.png` (SG description changed), `step1.7.png`, `step1.10.png`, `step6.2.png` (key file name in SSH command), `reference-architecture-lab3.png` (instance labels) |
| Lab 4 | `step1.1.png`, `step1.2.png`, `reference-architecture-final.png` (instance labels) |

### Orphaned files (flagged, not deleted)

- `reference-architecture-final.png` (repo root): a byte-identical duplicate of `aws-101-lab4/images/reference-architecture-final.png` that isn't referenced anywhere.
- `aws-101-lab2/images/step3.6.c.gif`: not referenced anywhere.

---

## TODOs needing verification

All TODOs are HTML comments (`<!-- TODO: … -->`) in the Markdown.

| File | Section | TODO |
| --- | --- | --- |
| `aws-101-lab2/README.md` | Step 1 — BYOL note | Verify current Marketplace listing titles and architectures (x86_64 / Arm64) |
| `aws-101-lab2/README.md` | Step 3.4 — Instance type | Verify `c5.large` against the current FortiGate-VM on AWS supported instance types |
| `aws-101-lab2/README.md` | Step 3.7 — Storage | Verify the default block-device mapping (root + log volume) of the current FortiGate BYOL AMI |
| `aws-101-lab2/README.md` | Step 3 — IMDSv2 callout | Verify IMDSv2-only support for the FortiOS version in use |
| `aws-101-lab2/README.md` | Additional Resources | Verify the FortiOS documentation version (8.0.0) used across the workshop |
| `aws-101-lab3/README.md` | Step 1.4 — Instance type | Verify Free Tier eligibility of `t3.micro` in `ca-central-1` |
| `aws-101-lab3/README.md` | Step 1.8 — IMDSv2 callout | Verify the Ubuntu 26.04 AMI defaults to IMDSv2-only |
| `aws-101-lab4/README.md` | NAT-T deep dive | Verify FortiOS behaviour when NAT is detected and NAT-T is disabled on one peer |
| `aws-101-lab4/README.md` | Step 5 — Tunnel status | Verify the IPsec monitor GUI path for the FortiOS version in use |
| `aws-101-lab4/README.md` | Troubleshooting — Tunnel flapping | Verify FortiOS default Phase 1 / Phase 2 key lifetimes |

---

## Final check

```bash
grep -rni "azure\|vnet\|nsg\|resource group\|arm template\|bicep" README.md aws-101-lab*/README.md
```

**Result:** zero matches for every search term except "resource group". The only matches are for **"resource group"**, and each one refers to the AWS-native **AWS Resource Groups** service (the tag-based group `redwood-aws101-lab-rg`, the *Resource Groups & Tag Editor* console, and the AWS docs link). These are allowed AWS-native exceptions:

- `aws-101-lab1/README.md`: prerequisites, What You'll Build, tagging section, Step 1 (heading, text, console navigation, alt text), validation, summary, checklist, Additional Resources
- `aws-101-lab4/README.md`: Clean-Up steps 1 and 8, plus the intro line

A secondary check, `grep -rni "az-101\|udr\|vnic\|nva\|hyperscaler"`, returns zero matches.
