# Cross-Region VPC Peering POC (Mumbai to Singapore)

A hands-on AWS proof of concept that connects an application server in one VPC to a private database tier in another VPC, in a different region, using **VPC peering**. The database side has no internet access at all.

**Status:** Phase 1 (peering, routing, connectivity tests) is complete. Phase 2 (RDS in the private subnet) is planned and documented below.

---

## 1. Objective

- Understand VPC peering concepts: CIDR planning, route tables, security groups, DNS, limitations.
- Build a production-style layout: a public application tier and a private data tier in separate VPCs.
- Prove private connectivity with ping, HTTP, SSH and file transfer over private IPs.
- Extend the setup by launching an RDS database in the private subnet and connecting to it from the public application server.

---

## 2. Architecture

```
                         Internet
                            |
                  [ Internet Gateway ]
                            |
   MUMBAI (ap-south-1)      |            SINGAPORE (ap-southeast-1)
 +--------------------------v--+        +------------------------------+
 | app-vpc01   10.0.0.0/24     |        | Db-Vpc   10.20.0.0/16        |
 |                             |        |                              |
 |  Public subnet 10.0.0.0/26  |        |  Private subnet 10.20.0.0/20 |
 |  [ poc-app-server ]  ------ | ==pcx==|  [ DB server ]  (Phase 1)    |
 |                             |        |  [ RDS ]        (Phase 2)    |
 +-----------------------------+        |  No IGW, no NAT              |
                                        +------------------------------+
        pcx-0f983a7a9d667cc8f  (inter-region VPC peering, private, encrypted)
```

| | App VPC | DB VPC |
|---|---|---|
| Name | `app-vpc01` | `Db-Vpc` |
| Region | Mumbai (`ap-south-1`) | Singapore (`ap-southeast-1`) |
| VPC ID | `vpc-076a897c96607b5db` | `vpc-023cc299cb387fb82` |
| CIDR | `10.0.0.0/24` | `10.20.0.0/16` |
| Subnet | `app-public-subnet01`, `10.0.0.0/26`, `ap-south-1a` | `db-private-subnet`, `10.20.0.0/20`, `ap-southeast-1c` |
| Route table | `app-Rt` | `rtb-0f126c80e270c7bab` |
| Internet Gateway | `app-vpc-igw` | None |
| Server | `poc-app-server` (public IP) | DB server (no public IP) |

---

## 3. What was built (Phase 1)

### 3.1 CIDR planning
The two VPCs must not overlap. The final plan:

| VPC | CIDR | Range |
|---|---|---|
| App | `10.0.0.0/24` | `10.0.0.0` - `10.0.0.255` |
| DB | `10.20.0.0/16` | `10.20.0.0` - `10.20.255.255` |

### 3.2 App VPC (Mumbai)
1. Created `app-vpc01` (`10.0.0.0/24`).
2. Created the public subnet `app-public-subnet01` (`10.0.0.0/26`) in `ap-south-1a`.
3. Created and attached the Internet Gateway `app-vpc-igw`.
4. Added the route `0.0.0.0/0` to the Internet Gateway in route table `app-Rt`.
5. Launched `poc-app-server` with a public IP and SSH allowed from my IP only.

### 3.3 DB VPC (Singapore)
1. Created `Db-Vpc` (`10.20.0.0/16`).
2. Created the private subnet `db-private-subnet` (`10.20.0.0/20`).
3. Kept the route table with only the `local` route.
4. Created **no** Internet Gateway and **no** NAT Gateway.
5. Disabled auto-assign public IP, so nothing in this VPC is reachable from the internet.

### 3.4 Peering connection
1. In Mumbai: created a peering request from `app-vpc01` to another region (Singapore), pasting the `Db-Vpc` VPC ID.
2. In Singapore: accepted the request. Status became **Active**.
3. Enabled DNS resolution on both sides of the peering.

Peering connection ID: `pcx-0f983a7a9d667cc8f`.

### 3.5 Route tables
Active status only means the link exists. Each side needs a route to the other VPC:

| Route table | Destination | Target |
|---|---|---|
| `app-Rt` (Mumbai) | `10.0.0.0/24` | local |
| `app-Rt` (Mumbai) | `0.0.0.0/0` | `igw-05d852df6a2ce4dc1` |
| `app-Rt` (Mumbai) | `10.20.0.0/16` | `pcx-0f983a7a9d667cc8f` |
| DB route table (Singapore) | `10.20.0.0/16` | local |
| DB route table (Singapore) | `10.0.0.0/24` | `pcx-0f983a7a9d667cc8f` |

Both directions are required: without the route on the DB side, requests arrive but replies cannot return.

### 3.6 Security group for the DB server (`db-app-sg`)

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | `10.0.0.0/24` | Login and file copy from the App VPC |
| Custom TCP | 8080 | `10.0.0.0/24` | Test service port |
| Custom ICMP (All) | n/a | `10.0.0.0/24` | Ping |

Only the App VPC range is allowed (least privilege). A CIDR is used instead of a security group ID because security groups cannot be referenced across regions.

### 3.7 DB server stand-in
The DB server was launched in the private subnet with no public IP. A user-data script starts a small listener on port 8080 to act as a service to test against:

```bash
#!/bin/bash
nohup python3 -m http.server 8080 --directory /tmp &
```

---

## 4. Test results

All tests were run from the App server using the DB server's **private IP**.

| Test | Command | Result |
|---|---|---|
| Ping | `ping 10.20.11.183` | Replies, about 60 ms (Mumbai to Singapore) |
| Ping | `ping -c 4 10.20.6.2` | 4/4 received, 0% loss |
| HTTP | `curl http://10.20.6.2:8080` | Directory listing returned |
| SSH | `ssh -i db-key.pem ec2-user@10.20.6.2` | Logged in to the DB server |
| File copy | `scp -i db-key.pem poc-folder/test.txt ec2-user@10.20.6.2:/tmp/` | 100% |
| File check | `curl http://10.20.6.2:8080/test.txt` | `hello from App server` |

Servers were relaunched during the POC, so two pairs appear in the screenshots: `10.0.0.34` to `10.20.11.183`, and `10.0.0.21` to `10.20.6.2`.

---

## 5. Problems faced and fixes

| Problem | Cause | Fix |
|---|---|---|
| First DB VPC was `10.0.0.0/16` | It overlaps the App VPC (`10.0.0.0/24`), so peering is impossible | A VPC's primary CIDR cannot be edited. Deleted it and recreated as `10.20.0.0/16` |
| Route target was Virtual Private Gateway | Wrong target type for peering | Changed the target to **Peering Connection** |
| App subnet auto-assign public IP is No | Subnet setting not enabled | Enabled public IP at instance launch |
| `Load key "db-key.pem": invalid format` | Pasting the key text damaged its line breaks | Copied the original `.pem` with `scp`. `ssh-keygen -y -f db-key.pem` then printed the public key and SSH worked |
| `scp: dest open "db-key.pem": Permission denied` | An old read-only copy of the key was in the way | Removed the old file, then copied again |
| `cat: poc-folder/test.txt: No such file` on the DB server | The folder had not been copied yet | Copied `test.txt` to `/tmp` and verified it |

---

## 6. Key concepts learned

- **Peering is not transitive.** A peered with B, and B peered with C, does not let A reach C.
- **No edge-to-edge routing.** A peered VPC cannot use the other VPC's Internet Gateway, NAT, VPN or Direct Connect.
- **CIDRs must not overlap**, and the primary CIDR cannot be changed after creation.
- **Active is not enough.** Routes on both sides and security group rules are also required.
- **Route problem vs security group problem.** A missing route and a missing rule both look like a timeout, so check each layer separately.
- **Cross-region peering** is encrypted and stays on the AWS backbone, but costs per GB and adds latency (about 60 ms here).
- **Security groups are stateful**, so replies to allowed requests are allowed automatically.

---

## 7. Phase 2 (planned): RDS in the private subnet

**Goal:** launch an RDS database in `Db-Vpc` with no public access, and connect to it from `poc-app-server` over the peering connection.

### 7.1 Prerequisites
- Peering connection Active, with routes on both sides (done).
- Enable **DNS resolution** on both sides of the peering connection (done), and **DNS hostnames** on both VPCs (recommended).

### 7.2 Add a second subnet (RDS requirement)
An RDS DB subnet group must span subnets in **at least 2 Availability Zones**, even for a single-AZ database.

| Item | Value |
|---|---|
| Subnet name | `db-private-subnet-b` |
| VPC | `Db-Vpc` |
| AZ | A different AZ from `db-private-subnet` (for example `ap-southeast-1a`) |
| CIDR | A non-overlapping range inside the VPC, for example `10.20.16.0/20` |
| Route table | The same private route table (local route only) |

### 7.3 Create the DB subnet group
**RDS, Subnet groups, Create:** name `poc-db-subnet-group`, VPC `Db-Vpc`, add both private subnets.

### 7.4 Create the RDS security group
Name `poc-rds-sg`, VPC `Db-Vpc`.

| Type | Port | Source |
|---|---|---|
| MySQL/Aurora | 3306 (or PostgreSQL 5432, MSSQL 1433) | `10.0.0.0/24` |

Use the App VPC CIDR, since the VPCs are in different regions.

### 7.5 Launch the database
**RDS, Create database:**

| Setting | Value |
|---|---|
| Engine | MySQL (or PostgreSQL) |
| Template | Free tier or Dev/Test |
| Instance class | Small (for example `db.t3.micro`) |
| VPC | `Db-Vpc` |
| DB subnet group | `poc-db-subnet-group` |
| **Public access** | **No** |
| Security group | `poc-rds-sg` |
| Multi-AZ | Off for the POC |

Copy the **endpoint** (a DNS name such as `mydb.xxxx.ap-southeast-1.rds.amazonaws.com`) once the status is Available.

### 7.6 Verify private DNS resolution
From the App server:

```bash
nslookup <rds-endpoint>
```

The answer should be a `10.20.x.x` address. This confirms the name resolves to the private IP over the peering.

### 7.7 Connect from the App server
Install the client (the App server has internet):

```bash
sudo dnf install -y mariadb105          # MySQL client on Amazon Linux 2023
mysql -h <rds-endpoint> -u <admin-user> -p
```

For PostgreSQL: `sudo dnf install -y postgresql15` and `psql -h <rds-endpoint> -U <admin-user> -d postgres`.

Quick check inside the client:

```sql
SELECT NOW();
CREATE DATABASE pocdb;
SHOW DATABASES;
```

### 7.8 Expected result
- The App server reaches RDS over private IPs through the peering connection.
- RDS has no public IP, and cannot be reached from the internet.
- Always use the **endpoint name**, not the IP, because the IP can change after a failover or restart.

### 7.9 Troubleshooting

| Symptom | Check |
|---|---|
| Connection times out | Route on both sides, `poc-rds-sg` allows the DB port from `10.0.0.0/24`, peering is Active |
| `nslookup` returns a public IP or nothing | DNS resolution on the peering connection, DNS hostnames on the VPCs |
| Cannot create the DB | The subnet group needs 2 AZs |
| `Access denied` | Wrong username or password (the network is fine) |

---

## 8. Optional extensions

- **VPC Flow Logs** on `Db-Vpc` to audit the peering traffic (`10.0.0.x` to `10.20.x.x`, `ACCEPT`).
- **Break it on purpose:** remove a route, then a security group rule, and observe the difference.
- **S3 Gateway Endpoint** in `Db-Vpc` so the private tier can reach S3 without internet (needs an IAM role on the instance).
- **Session Manager (SSM)** to log in to private servers without SSH keys.
- **Application Load Balancer** and **NAT Gateway** in the public subnet.
- **Same-region and cross-account peering** for comparison.
- **Terraform** version of the whole setup.

---

## 10. Security notes

- Keep SSH on the App server limited to your own IP.
- Do not leave private keys on servers. For real environments use SSH agent forwarding, a bastion host or Session Manager.
- Keep the DB tier private: no Internet Gateway, no public IP, and security group rules that allow only the application tier.
- Remove or blur account IDs in screenshots before sharing outside the team.
