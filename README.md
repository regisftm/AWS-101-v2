# AWS-101: FortiGate as AWS Cloud Firewall

## Fortinet Security Hands On Workshop | AWS Series

### Welcome

Organizations are rapidly adopting Amazon Web Services (AWS) to accelerate digital transformation, but this cloud migration introduces new security challenges. Extending consistent security policies from on-premises to the cloud while maintaining operational efficiency is critical. Many organizations struggle with fragmented security tools, rising costs from native cloud security services, and the complexity of securing hybrid environments. Without a unified approach, security teams face visibility gaps, manual processes, and difficulty maintaining compliance across their expanding attack surface.

In this workshop, participants will learn how to deploy FortiGate in AWS to protect their first cloud workloads. Using a realistic customer scenario, participants will build complete AWS networking infrastructure, deploy and license a FortiGate EC2 instance, configure security policies for north-south traffic inspection, and establish site-to-site IPsec VPN connectivity between AWS and on-premises environments.

At the heart of this solution is FortiGate as a Next-Generation Firewall (NGFW) in AWS, providing the same security capabilities and operational consistency that organizations rely on in their on-premises deployments. Participants will discover how to leverage AWS Route Tables to force traffic inspection, configure SNAT for internet access, and use FortiGate's VPN capabilities to replace AWS Site-to-Site VPN or AWS Transit Gateway for hybrid connectivity — delivering significant cost savings compared to AWS Network Firewall or AWS Gateway Load Balancer solutions.

### Time Requirements

The estimated time to complete this workshop is 3 hours.

### Target Audience

- Cloud security engineers and architects
- Network security professionals transitioning to cloud
- Fortinet administrators expanding to AWS
- IT professionals implementing enterprise security solutions
- Security consultants and presales engineers
- System administrators responsible for firewall management

**Experience Level**: Intermediate to advanced professionals with networking fundamentals and basic AWS knowledge.

### What You'll Learn

- Deploy complete AWS networking infrastructure (VPC, subnets, Internet Gateway, route tables)
- Configure FortiGate EC2 instance in AWS with BYOL licensing
- Implement AWS Route Tables to force traffic through FortiGate for inspection
- Disable Source/Destination check on FortiGate ENIs to enable transit
- Create firewall policies with SNAT for secure internet access
- Establish site-to-site IPsec VPN for hybrid connectivity
- Use FortiGate logs and FortiView for traffic visibility and troubleshooting
- Demonstrate cost savings and business value vs. AWS native security services

### Reference Architecture

After completing this bootcamp, you will have deployed the following architecture.

![reference-architecture](aws-101-lab4/images/reference-architecture-final.png)

## Laboratories

This workshop is organized in sequential laboratories. One lab will build up on top of the previous module, so please, follow the order as proposed below.

Lab 1 - [AWS Infrastructure Foundation](/aws-101-lab1/README.md)  
Lab 2 - [FortiGate EC2 Deployment & Traffic Steering](/aws-101-lab2/README.md)  
Lab 3 - [Security Policies & Traffic Testing](/aws-101-lab3/README.md)  
Lab 4 - [Site-to-Site VPN Configuration](/aws-101-lab4/README.md)

---

> [!NOTE]
> The workshop provides examples and sample code as instructional content for you to consume. These examples will help you understand how to configure Fortinet Security Fabric and build a functional solution. **Please note that these examples are not suitable for use in production environments**.

---

> [!CAUTION]
> If you are using an AWS account linked to your production environment, it would be more prudent not to use your "root user" account or an admin role with broad production access. Although the lab is designed to function in "isolated" mode, a "human" error when creating certain resources such as VPC peering and route tables could impact your production environment. **We recommend using an isolated AWS account or a dedicated sandbox account under AWS Organizations**.

---

> [!WARNING]
> This lab uses several EC2 instances. The entire lab should stay under **15** instances. At the end of the day, it will be important to terminate everything or at least stop the instances if you don't want any unpleasant surprises on your AWS bill.
