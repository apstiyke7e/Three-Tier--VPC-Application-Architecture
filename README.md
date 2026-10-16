# Three-Tier--VPC-Application-Architecture
 This repository will build the entire VPC networking infrastructure supporting a classic three-tier web application in the us-east-1 region using Availability Zones A and B (us-east-1a and us-east-1b)


Step-by-Step Implementation Guide

Step 1: Create the VPC  
1. Go to VPC Console → Your VPCs → Create VPC.  
2. Name: `Borderless-ThreeTier-VPC`  
3. IPv4 CIDR: `10.0.0.0/16`  
4. Tenancy: Default  
5. Create VPC.  


Step 2: Create Subnets (6 total) 
Create one by one (VPC Console → Subnets → Create subnet):  

- **Web-Public-A**: AZ us-east-1a, CIDR 10.0.1.0/24  
- **Web-Public-B**: AZ us-east-1b, CIDR 10.0.2.0/24  
- **App-Private-A**: AZ us-east-1a, CIDR 10.0.11.0/24  
- **App-Private-B**: AZ us-east-1b, CIDR 10.0.12.0/24  
- **DB-Private-A**: AZ us-east-1a, CIDR 10.0.21.0/24  
- **DB-Private-B**: AZ us-east-1b, CIDR 10.0.22.0/24  

For both **Web-Public** subnets only:  
Select each → Actions → Edit subnet settings → Check **Enable auto-assign public IPv4 address** → Save.

**Step 3: Create and Attach Internet Gateway**  
1. VPC Console → Internet Gateways → Create Internet Gateway.  
2. Name: `ThreeTier-IGW` → Create.  
3. Actions → Attach to VPC → Select your VPC → Attach.

**Step 4: Create Elastic IP for NAT Gateways**  
VPC Console → Elastic IPs → Allocate Elastic IP. Note the two EIP Allocation IDs.

**Step 5: Create NAT Gateways**  
1. VPC Console → NAT Gateways → Create NAT Gateway.  
2. **NAT-A**: Subnet = Web-Public-A, Elastic IP = first EIP, Name = `NAT-A`.  
3. **NAT-B**: Subnet = Web-Public-B, Elastic IP = second EIP, Name = `NAT-B`.  
Wait until both show **Available** (takes 2–5 minutes).

**Step 6: Create Route Tables**  
**Public Route Table** (for Web Tier):  
1. Route Tables → Create route table → Name: `Public-RT` → VPC: your VPC.  
2. Select it → Routes → Edit routes → Add route:  
   Destination `0.0.0.0/0` → Target = `ThreeTier-IGW`.  
3. Subnet associations → Edit → Associate **Web-Public-A** and **Web-Public-B**.

**Private Route Tables (2 for HA)**:  
1. Create `Private-RT` → Associate all four private subnets to this route table  
   Add route: `0.0.0.0/0` → Target = `NAT-GW`.  
.

**Step 7: Create Security Groups (Tier Isolation)**  
Security Groups → Create security group (in your VPC):  

- **Web-SG** (Web Tier + Bastion):  
  Inbound: SSH (22) from 0.0.0.0/0 , HTTP (80) from 0.0.0.0/0, ICMP (All) from 0.0.0.0/0.  

- **App-SG** (Application Tier):  
  Inbound: SSH (22) + ICMP from **Web-SG**, Custom TCP 8080 from **Web-SG**.  

- **DB-SG** (Database Tier):  
  Inbound: SSH (22) + ICMP from **App-SG**, Custom TCP 3306 from **App-SG**.  

**Step 8: Create Key Pair**  
EC2 Console → Key pairs → Create key pair → Name: `ThreeTier-Key` → .pem format → Download and `chmod 400 ThreeTier-Key.pem`.

**Step 9: Launch Placeholder EC2 VMs**  
EC2 Console → Launch instance (Amazon Linux 2023, t2.micro, same key pair for all):  

- **Bastion**: Subnet = Web-Public-A, Security Group = Web-SG (public IP auto).  
- **Web-VM**: Subnet = Web-Public-B, Security Group = Web-SG.  
- **App-VM**: Subnet = App-Private-A, Security Group = App-SG (no public IP).  
- **DB-VM**: Subnet = DB-Private-B, Security Group = DB-SG (no public IP).  

Wait until all four show **Running**. Note the private IPs and the two public IPs.

**Step 10: Test Connectivity (The Most Important Part!)**  

1. **Internet Access**  
   - SSH to Bastion and Web-VM (public IPs).  
   - Run: `curl ifconfig.me` and `ping 8.8.8.8` → Must succeed on public and private VMs.  

2. **Private Subnet Outbound (NAT Test)**  
   SSH to App-VM and DB-VM via Bastion (or jump through Web-VM) and repeat the curl/ping test.  

3. **Tier-to-Tier Connectivity**  
   - From Web-VM: `ping <App-VM-private-IP>` and `ping <DB-VM-private-IP>` (must succeed).  
   - From App-VM: `ping <DB-VM-private-IP>`.  
   - Test port reachability (no listener, so expect "Connection refused" — this proves SG allows traffic):  
     `timeout 3 bash -c "</dev/tcp/<App-private-IP>/8080" && echo "Port reachable" || echo "Timeout (SG blocked?)"`  

4. **Bastion Jump Test**  
   From my laptop i SSH to Bastion and from Bastion i SSH into App-VM then, the same for DB-VM.
   `ssh -i /path/to/key ec2-user@<App-private-IP>` 

5. **Security Validation**  
   - I couldn't SSH directly to App-VM or DB-VM private IPs from my  laptop → It failed (timeout).  
     

**Success Criteria**  
- All VMs can reach the internet.  
- Web tier can communicate with App tier.  
- App tier can communicate with DB tier.  
- Private instances have no direct inbound internet access.  
- Architecture spans two AZs with proper routing and isolation.
