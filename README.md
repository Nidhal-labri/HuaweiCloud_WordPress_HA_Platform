# 🌆 Huawei Cloud Project: Scalable WordPress Platform with Auto Scaling, Backup, Monitoring & Cross-VPC Migration

**A highly available WordPress website on Huawei Cloud — built with DDM + RDS, ELB and Auto Scaling, backup & restore, CES/LTS monitoring, and cross-VPC server migration via SMS and a NAT Gateway.**

This project simulates a real smart-city scenario: a WordPress website that must be **scalable**, **recoverable**, **auditable**, and **portable** across networks. The full production stack was deployed on Huawei Cloud, then layered with resilience (backup/restore + auto scaling), observability (alarms + log analytics), and a complete server migration from one VPC to another with internet access through a NAT gateway.

---

## 🌐 Why This Project?

This project brings together most of the core building blocks of a production cloud deployment in a single end-to-end architecture. Rather than deploying a single server, it builds a database middleware layer (DDM) in front of RDS, places the web tier behind an Elastic Load Balancer with an Auto Scaling Group, protects it with scheduled and manual backups, monitors it with Cloud Eye alarms and Log Tank log analytics, and finally migrates the live server across VPCs using VPC peering, Server Migration Service, and a NAT gateway. It’s a solid demonstration of how availability, cost-awareness, security, and operability fit together on Huawei Cloud.

---

## 🗺️ Architecture Diagram

<img width="1286" height="727" alt="image" src="https://github.com/user-attachments/assets/e5afcb8f-bc9c-4806-bbbc-f62ccf3b9a2c" />

---

## 🧱 Key Huawei Cloud Services Used

- **ECS (Elastic Cloud Server)** – Hosts the WordPress web server and the migration target.
- **DDM (Distributed Database Middleware)** – Database middleware layer in front of RDS.
- **RDS (Relational Database Service)** – Managed database backing the WordPress site.
- **ELB (Elastic Load Balance)** – Distributes web traffic across the scaling instances.
- **AS (Auto Scaling)** – Automatically adjusts instance count based on CPU load.
- **CBR / CSBS (Cloud Backup & Recovery)** – Scheduled and manual ECS backups.
- **IMS (Image Management Service)** – Private image created from a backup.
- **CES (Cloud Eye)** – Host monitoring and CPU alarm rules.
- **LTS (Log Tank Service)** – Log ingestion, search, and transfer to OBS.
- **OBS (Object Storage Service)** – Long-term storage of transferred logs.
- **VPC + VPC Peering** – Network isolation and inter-VPC connectivity.
- **SMS (Server Migration Service)** – Migrates the live server across VPCs.
- **NAT Gateway + EIP** – Inbound (DNAT) and outbound (SNAT) internet access.

---

## 🛠️ Tasks

### 🚀 Task 1 – Deploying a WordPress Website Production Environment

#### Subtask 1: Buy a DDM instance
Provisioned a **Distributed Database Middleware (DDM)** instance named `ddm-wordpress` inside `vpc-wordpress` / `subnet-wordpress`, attached to `sg-wordpress`. Rather than pointing WordPress straight at RDS, DDM sits in front of the database as a middleware layer — decoupling the application from the underlying instance so the data tier can later be sharded or scaled without touching the site configuration.

**📷 `1-1-1ddm`** — <img width="1179" height="911" alt="1-1-1ddm" src="https://github.com/user-attachments/assets/e4ff6b4b-7002-4aa3-ae24-eb2b01b43ef5" />

#### Subtask 2: Create a schema and an account
Created an **unsharded schema** `db_wordpress` on `ddm-wordpress`, bound to the pre-provisioned `rds-wordpress` RDS instance, and confirmed it reached the **Running** state.

**📷 `1-2-1schema`** — *(paste screenshot here)*

Then created a DDM account (`ICT_user##`) with full permissions on that schema.

**📷 `1-2-2account`** — *(paste screenshot here)*

#### Subtask 3: Create an ECS to access WordPress
Launched an ECS named `wordpress-server` (General computing-plus, 2 vCPUs | 4 GB, x86) from the shared full-ECS image **Wordpress_for_ICT**, in `vpc-wordpress` with `sg-wordpress` and an auto-assigned EIP.

**📷 `1-3-1wordpress-server`** — *(paste screenshot here)*

Browsed to `http://<EIP>/wordpress` to reach the WordPress setup page.

**📷 `1-3-2wordpress-setup`** — *(paste screenshot here)*

#### Subtask 4: Configure the WordPress database
Pointed the WordPress install wizard at the DDM layer — using the schema name as the database, the DDM account and password, and the **DDM connection address** as the database host.

**📷 `1-4-1ddm configuration`** — *(paste screenshot here)*

Ran the installation and logged in, completing the production WordPress environment.

**📷 `1-4-2wordpress`** — *(paste screenshot here)*

---

### 💾 Task 2 – Backing Up the ECS

#### Subtask 1: Configure an auto backup policy
Purchased a **Cloud Server Backup vault** (`vault-wordpress`, 100 GB) and created an auto backup policy (`policy-auto`) running weekly on Sundays at **07:00** with a **1-month retention**.

**📷 `2-1-1policy-auto`** — *(paste screenshot here)*

Applied `policy-auto` to the vault and associated `wordpress-server` with it.

**📷 `2-1-2vault-wordpress`** — *(paste screenshot here)*

#### Subtask 2: Manually back up data
Ran a **manual backup** (`manualbk-vault-wordpress`) of `wordpress-server` to capture an on-demand recovery point.

**📷 `2-2-1manualbk-vault`** — *(paste screenshot here)*

#### Subtask 3: Create an image using backup
Used the manual backup to build a **full-ECS private image** named `image-from-backup`. Capturing the already-configured server as an image means every instance the Auto Scaling Group launches later comes up as an exact, ready-to-serve replica — no manual setup on new machines.

**📷 `2-3-1image-from-backup`** — *(paste screenshot here)*

#### Subtask 4: Restore the ECS using backup
Restored `wordpress-server` from the manual backup and confirmed the restore task completed successfully.

**📷 `2-4-1tasks`** — *(paste screenshot here)*

---

### ⚖️ Task 3 – Configuring Website Load Balancing and Auto Scaling

#### Subtask 1: Create an ELB load balancer
Unbound the EIP from `wordpress-server` so the instance is no longer directly reachable from the internet. With the public entry point moved to the load balancer, all traffic is forced through the ELB — a prerequisite for scaling the web tier behind a single stable address.

**📷 `3-1-1wordpress-server-noeip`** — *(paste screenshot here)*

Created a **shared, public ELB** (`elb-wordpress`) in `vpc-wordpress` with a new EIP.

**📷 `3-1-2elb-wordpress`** — *(paste screenshot here)*

Added a **TCP:80 listener** (`listener-wordpress`).

**📷 `3-1-3listener`** — *(paste screenshot here)*

Configured a backend server group (`server_group-wordpress`) and registered `wordpress-server` on port 80.

**📷 `3-1-4server-group`** — *(paste screenshot here)*

Verified access through `http://<ELB EIP>/wordpress`.

**📷 `3-1-5elb-wordpress`** — *(paste screenshot here)*

#### Subtask 2: Create an AS group and configure scaling rules
Created an **AS configuration** (`as-config-wordpress`) from the `image-from-backup` private image — closing the loop from Task 2, so scaled-out instances inherit the fully configured WordPress setup instead of a blank OS.

**📷 `3-2-1as-config-wordpress`** — *(paste screenshot here)*

Created an **AS group** (`as-group-wordpress`) with min **1** / desired **1** / max **4**, linked to `elb-wordpress` and `server_group-wordpress`.

**📷 `3-2-2as-group-wordpress`** — *(paste screenshot here)*

Manually added `wordpress-server` to the group and verified the instance details.

**📷 `3-2-3as-instances`** — *(paste screenshot here)*

Added a scale-out policy (`as-policy-expanding`) that adds **1 instance when CPU usage ≥ 40%** (5-min interval, 120s cooldown).

**📷 `3-2-4as-policy-details`** — *(paste screenshot here)*

#### Subtask 3: Verify the auto scaling
Validated scaling by manually setting **expected instances to 4**, then back to **2**, and executing `as-policy-expanding` manually — then reviewed the resulting scaling actions.

**📷 `3-3-1scaling-actions`** — *(paste screenshot here)*

Waited for the changes to take effect and captured the monitoring curves.

**📷 `3-3-2as-group-web-monitoring`** — *(paste screenshot here)*

---

### 🔍 Task 4 – Monitoring Websites and Logs

#### Subtask 1: Install the Agent (monitoring plug-in)
Installed the **CES Agent** on the AS-created ECS instances and confirmed under **Cloud Eye → Server Monitoring** that each Agent showed as **Running**.

**📷 `4-1-1installed-agent`** — *(paste screenshot here)*

#### Subtask 2: Configure server monitoring alarm rules
Created CPU-usage alarm rules (`alarm-rule1` / `alarm-rule2`) on the scaling instances — triggering at **≥ 40% over 3 consecutive 5-minute periods**.

**📷 `4-2-1alarm-rules`** — *(paste screenshot here)*

Generated CPU load on the servers to trip the alarms and confirmed the status changed to **Alarm** in the records.

**📷 `4-2-2history alarm`** — *(paste screenshot here)*

#### Subtask 3: Configure a log monitoring policy for ECSs
Created a log group `lts-group-wordpress` (7-day retention) and a log stream `lts-topic-wordpress`.

**📷 `4-3-1Log Stream`** — *(paste screenshot here)*

Installed **ICAgent** on `wordpress-server` via AK/SK and confirmed it appeared on the **Hosts** tab.

**📷 `4-3-2ICAgent Hosts`** — *(paste screenshot here)*

Configured host-group ingestion of `/var/log/messages` (`collect_wordpress`, single-line, system time).

**📷 `4-3-3collect-wordpress`** — *(paste screenshot here)*

Searched the ingested logs for `successful` to confirm collection was working.

**📷 `4-3-4realtime log`** — *(paste screenshot here)*

#### Subtask 4: Transfer logs to OBS
Created a **log transfer rule** from `lts-group-wordpress` to the `obs-result-ict##` bucket (raw log format, 2-minute interval, prefix `log-wordpress`).

**📷 `4-4-1logs-transfer-config`** — *(paste screenshot here)*

Confirmed the log folder was automatically uploaded to the OBS bucket.

**📷 `4-4-2logs-in-bucket`** — *(paste screenshot here)*

---

### 🔀 Task 5 – Migrating Hosts Across VPCs

#### Subtask 1: Create a VPC and a security group
Created `vpc-migration` (`10.1.0.0/16`, with `subnet-migration` `10.1.0.0/24`) in the Hong Kong region.

**📷 `5-1-1vpc-migration`** — *(paste screenshot here)*

Created a security group `sg-migration` using the **General-purpose web server** template with **TCP port 5066** manually allowed.

**📷 `5-1-2sg-migration`** — *(paste screenshot here)*

#### Subtask 2: Create a VPC peering connection
Created a **VPC peering connection** (`peering-wordpress`) between `vpc-wordpress` (local) and `vpc-migration` (peer).

**📷 `5-2-1peering-wordpress`** — *(paste screenshot here)*

A peering connection only opens the door — traffic still needs routes on both sides to actually cross. Added the **local route** pointing to the `vpc-migration` CIDR via `peering-wordpress`.

**📷 `5-2-2local route`** — *(paste screenshot here)*

Added the matching **peer route** back to the `vpc-wordpress` CIDR, so return traffic has a path home and the two VPCs can communicate in both directions.

**📷 `5-2-3peer route`** — *(paste screenshot here)*

#### Subtask 3: Create a target ECS
Launched the migration target ECS `ecs-migration` (2 vCPUs | 4 GB, CentOS 7.6, Pay-per-use) inside `vpc-migration` / `sg-migration`.

**📷 `5-3-1wordpress-server`** — *(paste screenshot here)*

#### Subtask 4: Migrate wordpress-server to another VPC
Re-bound an EIP to `wordpress-server`, installed the **SMS migration Agent** (entering AK/SK), and confirmed the "sms agent start up successfully!" message.

**📷 `5-4-1agent`** — *(paste screenshot here)*

Configured the target as the existing `ecs-migration` (Linux file-level migration, private network, no partition resizing) and started the migration.

**📷 `5-4-2migration task`** — *(paste screenshot here)*

#### Subtask 5: Configure a NAT gateway
Created a public **NAT gateway** (`NATGW-HK`) in `vpc-migration` with two EIPs (`bandwidth-hk1` / `bandwidth-hk2`). Added a **DNAT rule** (using `bandwidth-hk2`, all ports) to map the public EIP onto the private instance — this handles *inbound* traffic, letting external users reach `ecs-migration` even though it has no EIP of its own.

**📷 `5-5-1DNAT-RULE`** — *(paste screenshot here)*

Added an **SNAT rule** (using `bandwidth-hk1`) for the opposite direction — *outbound* traffic, so the private instance can initiate connections to the internet (updates, pings) behind a shared public IP.

**📷 `5-5-2SNAT-RULE`** — *(paste screenshot here)*

Verified **inbound** access via `http://<bandwidth-hk2 EIP>/wordpress`.

**📷 `5-5-3migration`** — *(paste screenshot here)*

Verified **outbound** access by pinging `www.huaweicloud.com` from `ecs-migration`.

**📷 `5-5-4ping result`** — *(paste screenshot here)*

---

## ✅ Outcome

The final environment runs a WordPress site on a DDM + RDS backend, fronted by an ELB and backed by an Auto Scaling Group that adds capacity under load. It is protected by scheduled and manual backups (with restore verified), monitored through Cloud Eye alarms and LTS log analytics exported to OBS, and fully migratable across VPCs with internet access provided by a NAT gateway — a complete, resilient, and observable cloud deployment.

---

## 🧠 Skills Demonstrated

Compute & scaling (ECS, ELB, AS) · Managed databases (DDM, RDS) · Backup & recovery (CBR, IMS) · Observability (CES, LTS, OBS) · Networking (VPC, VPC peering, NAT gateway, EIP) · Server migration (SMS).

---

> ℹ️ *This project is my own hands-on implementation based on the 2021–2022 Huawei ICT Competition (Cloud Track) Global Final scenario. It documents the work I performed; it is not an official solution.*

---

## ✍️ Author

**Made with 💻 by Nidhal Labri**
🔗 [LinkedIn](https://www.linkedin.com/in/nidhal-labri/)
