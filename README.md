# 🌆 Huawei Cloud Project: Scalable WordPress Platform with Auto Scaling, Backup, Monitoring & Cross-VPC Migration

**A highly available WordPress website on Huawei Cloud — built with DDM + RDS, ELB and Auto Scaling, backup & restore, CES/LTS monitoring, and cross-VPC server migration via SMS and a NAT Gateway.**

This project simulates a real smart-city scenario: a WordPress website that must be **scalable**, **recoverable**, **auditable**, and **portable** across networks. The full production stack was deployed on Huawei Cloud, then layered with resilience (backup/restore + auto scaling), observability (alarms + log analytics), and a complete server migration from one VPC to another with internet access through a NAT gateway.

---

## 🌐 Why This Project?

This project brings together most of the core building blocks of a production cloud deployment in a single end-to-end architecture. Rather than deploying a single server, it builds a database middleware layer (DDM) in front of RDS, places the web tier behind an Elastic Load Balancer with an Auto Scaling Group, protects it with scheduled and manual backups, monitors it with Cloud Eye alarms and Log Tank log analytics, and finally migrates the live server across VPCs using VPC peering, Server Migration Service, and a NAT gateway. It’s a solid demonstration of how availability, cost-awareness, security, and operability fit together on Huawei Cloud.

---

## 🗺️ Architecture Diagram

<img width="1294" height="726" alt="image" src="https://github.com/user-attachments/assets/2a0953bd-0364-4e81-aeb8-82ed012321dc" />

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

**📷 `1-2-1schema`** — <img width="1920" height="912" alt="1-2-1schema" src="https://github.com/user-attachments/assets/b850ec22-7572-45c7-a2d7-f154cda273a1" />

Then created a DDM account (`ICT_user##`) with full permissions on that schema.

**📷 `1-2-2account`** — <img width="1920" height="912" alt="1-2-2account" src="https://github.com/user-attachments/assets/587098cb-b8bb-4a66-b77f-f4fdc17a5c23" />

#### Subtask 3: Create an ECS to access WordPress
Launched an ECS named `wordpress-server` (General computing-plus, 2 vCPUs | 4 GB, x86) from the shared full-ECS image **Wordpress_for_ICT**, in `vpc-wordpress` with `sg-wordpress` and an auto-assigned EIP.

**📷 `1-3-1wordpress-server`** — <img width="1920" height="912" alt="1-3-1wordpress-server" src="https://github.com/user-attachments/assets/25bfaad0-ce05-4269-a5de-4dd16c33b6d1" />

Browsed to `http://<EIP>/wordpress` to reach the WordPress setup page.

**📷 `1-3-2wordpress-setup`** — <img width="1179" height="962" alt="1-3-2wordpress-setup" src="https://github.com/user-attachments/assets/cb6af9f4-8b68-46da-92b2-e9b3a70087fd" />

#### Subtask 4: Configure the WordPress database
Pointed the WordPress install wizard at the DDM layer — using the schema name as the database, the DDM account and password, and the **DDM connection address** as the database host.

**📷 `1-4-1ddm configuration`** — <img width="1179" height="964" alt="1-4-1ddm-configuration" src="https://github.com/user-attachments/assets/c855ee33-d371-41d2-8db3-72e1bb3128ba" />


Ran the installation and logged in, completing the production WordPress environment.

**📷 `1-4-2wordpress`** — <img width="1179" height="965" alt="1-4-2wordpress" src="https://github.com/user-attachments/assets/44b10c73-78fb-4803-8be3-b4225d7242ec" />


---

### 💾 Task 2 – Backing Up the ECS

#### Subtask 1: Configure an auto backup policy
Purchased a **Cloud Server Backup vault** (`vault-wordpress`, 100 GB) and created an auto backup policy (`policy-auto`) running weekly on Sundays at **07:00** with a **1-month retention**.

**📷 `2-1-1policy-auto`** — <img width="1179" height="911" alt="2-1-1policy-auto" src="https://github.com/user-attachments/assets/2a1cfbfc-1ce6-4e29-9754-68dca1c5b049" />


Applied `policy-auto` to the vault and associated `wordpress-server` with it.

**📷 `2-1-2vault-wordpress`** — <img width="1179" height="911" alt="2-1-2vault-wordpress" src="https://github.com/user-attachments/assets/022a9790-68e5-467d-b2f8-dca23e367405" />


#### Subtask 2: Manually back up data
Ran a **manual backup** (`manualbk-vault-wordpress`) of `wordpress-server` to capture an on-demand recovery point.

**📷 `2-2-1manualbk-vault`** — <img width="1179" height="911" alt="2-2-1manualbk-vault" src="https://github.com/user-attachments/assets/0e543333-d41d-4559-9658-9a62d835aa3e" />


#### Subtask 3: Create an image using backup
Used the manual backup to build a **full-ECS private image** named `image-from-backup`. Capturing the already-configured server as an image means every instance the Auto Scaling Group launches later comes up as an exact, ready-to-serve replica — no manual setup on new machines.

**📷 `2-3-1image-from-backup`** — <img width="1179" height="911" alt="2-3-1image-from-backup" src="https://github.com/user-attachments/assets/f42659dd-c669-4259-90ee-2feb524d849d" />


#### Subtask 4: Restore the ECS using backup
Restored `wordpress-server` from the manual backup and confirmed the restore task completed successfully.

**📷 `2-4-1tasks`** — <img width="1179" height="466" alt="2-4-1tasks" src="https://github.com/user-attachments/assets/e8b50042-ade7-402c-8401-6406c7a03fb9" />


---

### ⚖️ Task 3 – Configuring Website Load Balancing and Auto Scaling

#### Subtask 1: Create an ELB load balancer
Unbound the EIP from `wordpress-server` so the instance is no longer directly reachable from the internet. With the public entry point moved to the load balancer, all traffic is forced through the ELB — a prerequisite for scaling the web tier behind a single stable address.

**📷 `3-1-1wordpress-server-noeip`** — <img width="1179" height="911" alt="3-1-1wordpress-server-noeip" src="https://github.com/user-attachments/assets/b0506082-d578-4135-a5db-c4ed1202139b" />


Created a **shared, public ELB** (`elb-wordpress`) in `vpc-wordpress` with a new EIP.

**📷 `3-1-2elb-wordpress`** — <img width="1920" height="912" alt="3-1-2elb-wordpress" src="https://github.com/user-attachments/assets/232d0965-cef3-49c4-a38f-72735f05dc56" />


Added a **TCP:80 listener** (`listener-wordpress`).

**📷 `3-1-3listener`** — <img width="1920" height="912" alt="3-1-3listener" src="https://github.com/user-attachments/assets/d227ca54-024c-4ec8-80dc-1c4c46efdb78" />


Configured a backend server group (`server_group-wordpress`) and registered `wordpress-server` on port 80.

**📷 `3-1-4server-group`** — <img width="1920" height="635" alt="3-1-4server group" src="https://github.com/user-attachments/assets/faab113b-3f50-44a9-83df-ef713b7c3c36" />


Verified access through `http://<ELB EIP>/wordpress`.

**📷 `3-1-5elb-wordpress`** — <img width="1920" height="968" alt="3-1-5elb-wordpress" src="https://github.com/user-attachments/assets/bc27e091-320e-487d-9305-4ad3681571e0" />


#### Subtask 2: Create an AS group and configure scaling rules
Created an **AS configuration** (`as-config-wordpress`) from the `image-from-backup` private image — closing the loop from Task 2, so scaled-out instances inherit the fully configured WordPress setup instead of a blank OS.

**📷 `3-2-1as-config-wordpress`** — <img width="1920" height="912" alt="3-2-1as-config-wordpress" src="https://github.com/user-attachments/assets/cd413d19-9d16-4a9d-b5ec-940c7f832b49" />


Created an **AS group** (`as-group-wordpress`) with min **1** / desired **1** / max **4**, linked to `elb-wordpress` and `server_group-wordpress`.

**📷 `3-2-2as-group-wordpress`** — <img width="1920" height="912" alt="3-2-2as-group-wordpress" src="https://github.com/user-attachments/assets/67ef4d15-3ac3-4300-a587-693fbe39dc10" />


Manually added `wordpress-server` to the group and verified the instance details.

**📷 `3-2-3as-instances`** — <img width="1920" height="912" alt="3-2-3as-instances" src="https://github.com/user-attachments/assets/363af9a8-1b54-4e42-b0e8-a9649bd22c30" />


Added a scale-out policy (`as-policy-expanding`) that adds **1 instance when CPU usage ≥ 40%** (5-min interval, 120s cooldown).

**📷 `3-2-4as-policy-details`** — <img width="1920" height="912" alt="3-2-4as-policy-details" src="https://github.com/user-attachments/assets/50afc3aa-36b4-4c2e-a520-c926a9f460fd" />


#### Subtask 3: Verify the auto scaling
Validated scaling by manually setting **expected instances to 4**, then back to **2**, and executing `as-policy-expanding` manually — then reviewed the resulting scaling actions.

**📷 `3-3-1scaling-actions`** — <img width="1920" height="547" alt="3-3-1scaling-actions" src="https://github.com/user-attachments/assets/7e8035f2-d399-43a1-b327-b5aaab2f7edc" />


Waited for the changes to take effect and captured the monitoring curves.

**📷 `3-3-2as-group-web-monitoring`** — <img width="1920" height="912" alt="3-3-2as-group-web-monitoring" src="https://github.com/user-attachments/assets/118df894-c806-4c5c-9875-b9e7b123e2ca" />


---

### 🔍 Task 4 – Monitoring Websites and Logs

#### Subtask 1: Install the Agent (monitoring plug-in)
Installed the **CES Agent** on the AS-created ECS instances and confirmed under **Cloud Eye → Server Monitoring** that each Agent showed as **Running**.

**📷 `4-1-1installed-agent`** — <img width="1179" height="476" alt="4-1-1installed-agent" src="https://github.com/user-attachments/assets/b0a44442-85c1-42dc-a47a-1a17354c16eb" />


#### Subtask 2: Configure server monitoring alarm rules
Created CPU-usage alarm rules (`alarm-rule1` / `alarm-rule2`) on the scaling instances — triggering at **≥ 40% over 3 consecutive 5-minute periods**.

**📷 `4-2-1alarm-rules`** — <img width="1179" height="911" alt="4-2-1alarm rules" src="https://github.com/user-attachments/assets/8fcdc066-c23c-4981-b682-f015eb035755" />


Generated CPU load on the servers to trip the alarms and confirmed the status changed to **Alarm** in the records.

**📷 `4-2-2history alarm`** — <img width="1920" height="499" alt="4-2-2history alarm" src="https://github.com/user-attachments/assets/5a7f9212-9a16-4cd5-ba59-e3ec91ddba5a" />


#### Subtask 3: Configure a log monitoring policy for ECSs
Created a log group `lts-group-wordpress` (7-day retention) and a log stream `lts-topic-wordpress`.

**📷 `4-3-1Log Stream`** — <img width="897" height="423" alt="4-3-1log stream" src="https://github.com/user-attachments/assets/f265fa09-ead2-43cd-abf8-5fd5d192bddb" />


Installed **ICAgent** on `wordpress-server` via AK/SK and confirmed it appeared on the **Hosts** tab.

**📷 `4-3-2ICAgent Hosts`** — <img width="1920" height="352" alt="4-3-2ICAgent-Hosts" src="https://github.com/user-attachments/assets/e608ee8c-f140-47c4-85d6-1634cd94ea90" />


Configured host-group ingestion of `/var/log/messages` (`collect_wordpress`, single-line, system time).

**📷 `4-3-3collect-wordpress`** — <img width="1920" height="912" alt="4-3-3collect-wordpress" src="https://github.com/user-attachments/assets/6e977726-9622-438d-9101-40dc27b57bbf" />


Searched the ingested logs for `successful` to confirm collection was working.

**📷 `4-3-4realtime log`** — <img width="1920" height="912" alt="4-3-4realtime log" src="https://github.com/user-attachments/assets/9d734be1-54e0-476a-b089-fb42998e69de" />


#### Subtask 4: Transfer logs to OBS
Created a **log transfer rule** from `lts-group-wordpress` to the `obs-result-ict##` bucket (raw log format, 2-minute interval, prefix `log-wordpress`).

**📷 `4-4-1logs-transfer-config`** — <img width="1920" height="912" alt="4-4-1logs-transfer-config" src="https://github.com/user-attachments/assets/fc479297-7352-4089-aa75-b6283999ebde" />


Confirmed the log folder was automatically uploaded to the OBS bucket.

**📷 `4-4-2logs-in-bucket`** — <img width="1920" height="912" alt="4-4-2logs-in-bucket" src="https://github.com/user-attachments/assets/41ba44a0-b566-41d4-8a51-015bc526da4b" />


---

### 🔀 Task 5 – Migrating Hosts Across VPCs

#### Subtask 1: Create a VPC and a security group
Created `vpc-migration` (`10.1.0.0/16`, with `subnet-migration` `10.1.0.0/24`) in the Hong Kong region.

**📷 `5-1-1vpc-migration`** — <img width="1221" height="1018" alt="5-1-1vpc-migration" src="https://github.com/user-attachments/assets/004aacdf-6498-4479-b9dc-384fae462961" />


Created a security group `sg-migration` using the **General-purpose web server** template with **TCP port 5066** manually allowed.

**📷 `5-1-2sg-migration`** — <img width="1920" height="912" alt="5-1-2sg-migration" src="https://github.com/user-attachments/assets/cea8d446-cf3d-410a-9fe9-0c29ef720548" />


#### Subtask 2: Create a VPC peering connection
Created a **VPC peering connection** (`peering-wordpress`) between `vpc-wordpress` (local) and `vpc-migration` (peer).

**📷 `5-2-1peering-wordpress`** — <img width="1920" height="912" alt="5-2-1peering-wordpress" src="https://github.com/user-attachments/assets/e6818989-1d9a-40e0-825f-b9995eeb0469" />


A peering connection only opens the door — traffic still needs routes on both sides to actually cross. Added the **local route** pointing to the `vpc-migration` CIDR via `peering-wordpress`. Added the matching **peer route** back to the `vpc-wordpress` CIDR, so return traffic has a path home and the two VPCs can communicate in both directions.

**📷 `5-2-2local route`** — <img width="1127" height="545" alt="5-2-2SNAT-RULE" src="https://github.com/user-attachments/assets/5435770f-018b-4e7d-8825-9635e51e0f59" />


#### Subtask 3: Create a target ECS
Launched the migration target ECS `ecs-migration` (2 vCPUs | 4 GB, CentOS 7.6, Pay-per-use) inside `vpc-migration` / `sg-migration`.

**📷 `5-3-1wordpress-server`** — <img width="1920" height="912" alt="5-3-1wordpress-server" src="https://github.com/user-attachments/assets/ae032c99-6b62-4943-bdfa-2947860bf78f" />


#### Subtask 4: Migrate wordpress-server to another VPC
Re-bound an EIP to `wordpress-server`, installed the **SMS migration Agent** (entering AK/SK), and confirmed the "sms agent start up successfully!" message. Configured the target as the existing `ecs-migration` (Linux file-level migration, private network, no partition resizing) and started the migration.

**📷 `5-4-1agent`** — <img width="1920" height="964" alt="5-4-1agent" src="https://github.com/user-attachments/assets/758c4281-8a80-40f6-80ff-eb93e6561d35" />


#### Subtask 5: Configure a NAT gateway
Created a public **NAT gateway** (`NATGW-HK`) in `vpc-migration` with two EIPs (`bandwidth-hk1` / `bandwidth-hk2`). Added a **DNAT rule** (using `bandwidth-hk2`, all ports) to map the public EIP onto the private instance — this handles *inbound* traffic, letting external users reach `ecs-migration` even though it has no EIP of its own.

**📷 `5-5-1DNAT-RULE`** — <img width="1127" height="903" alt="5-5-1DNAT-RULE" src="https://github.com/user-attachments/assets/7622a42a-e1de-4493-971b-57f33d06cd69" />


Added an **SNAT rule** (using `bandwidth-hk1`) for the opposite direction — *outbound* traffic, so the private instance can initiate connections to the internet (updates, pings) behind a shared public IP.

**📷 `5-5-2SNAT-RULE`** — <img width="1121" height="911" alt="5-5-2DNAT-RULE" src="https://github.com/user-attachments/assets/2b90375a-eb72-4e87-86bf-40b73595ffc3" />


Verified **inbound** access via `http://<bandwidth-hk2 EIP>/wordpress`.

**📷 `5-5-3migration`** — <img width="1127" height="953" alt="5-5-3migration" src="https://github.com/user-attachments/assets/b5e664a3-5dd5-4e1a-8ba3-90c2150de07d" />


Verified **outbound** access by pinging `www.huaweicloud.com` from `ecs-migration`.

**📷 `5-5-4ping result`** — <img width="1127" height="903" alt="5-5-4ping-result" src="https://github.com/user-attachments/assets/a20d42ce-b9aa-4163-af8f-17973c585932" />


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
